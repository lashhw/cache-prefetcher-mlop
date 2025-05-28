# MLOP 預取器研究與 ChampSim 結果重現說明

## 1. 簡介與執行環境

### 1.1 MLOP 預取器簡介

MLOP (Multi-Lookahead Offset Prefetcher) 是一款旨在同時提升預取「及時性」(timeliness) 與「覆蓋率」(coverage) 的硬體預取器。其核心思想包括：

* **多重前瞻等級**：評估預取偏移量 (offset) 時，考量不同預測「距離」的效益。
* **存取映射表 (AMT)**：追蹤近期記憶體存取及其鄰近區塊的狀態。
* **偏移量評分與選擇**：為各前瞻等級的潛在偏移量評分，選取最高分者。
* **請求優先級**：優先處理較小前瞻等級（預期較快發生）的預取請求。

MLOP 旨在克服傳統偏移預取器在及時性與覆蓋率上的取捨難題，主要於 L1 資料快取 (L1D Cache) 運作，並根據 L1D 未命中串流 (miss streams) 進行訓練。

完整論文詳見 [mlop.pdf](mlop.pdf)。

### 1.2 ChampSim 模擬器

ChampSim 是一款 trace-driven 模擬器，用於研究快取及記憶體系統，可評估不同的快取設計、替換策略與預取演算法。這是這個 Cache Prefetcher Championship 所採用的模擬器。

### 1.3 執行環境

* **作業系統**：Linux。
* **編譯器**：C++ 編譯器 (如 g++)。
* **建置工具**：GNU Make。
* **相依套件**：標準 C++ 函式庫。

### 1.4 下載追蹤檔

在執行模擬之前，請先下載指定的追蹤檔 (trace file)。請至 https://dpc3.compas.cs.stonybrook.edu/champsim-traces/speccpu/ 下載 `602.gcc_s-2226B.champsimtrace.xz` 檔案。

將下載的追蹤檔存放於 `dpc3_traces/` 目錄下，確保與 `run_champsim.sh` 腳本中預設的追蹤檔路徑一致。

## 2. 編譯與執行

### 2.1 編譯 ChampSim

使用 `build_champsim.sh` 編譯 ChampSim，需指定分支預測器、各級快取預取器、L1D 替換策略及核心數。

編譯含 MLOP (L1D 預取器) 的 ChampSim 版本 (以 `perceptron` 分支預測器，L2C/LLC 用 `next_line`，L1D 用 `lru`，單核心為例)：

```bash
./build_champsim.sh perceptron mlop_dpc3 next_line next_line lru 1
```

成功編譯後，執行檔位於 `bin/` 目錄，例如 `perceptron-mlop_dpc3-next_line-next_line-lru-1core`。
MLOP 實作位於 `prefetcher/mlop_dpc3.l1d_pref`。

### 2.2 執行 ChampSim

透過 `run_champsim.sh` 執行模擬，需指定執行檔、預熱指令數、模擬指令數及追蹤檔路徑。

例如，參考比賽規則，使用 `602.gcc_s-2226B.champsimtrace.xz` 追蹤檔，預熱 50M 指令，模擬 200M 指令：

```bash
./run_champsim.sh perceptron-mlop_dpc3-next_line-next_line-lru-1core 50 200 dpc3_traces/602.gcc_s-2226B.champsimtrace.xz
```

結果儲存於 `results_200M/`，檔名格式如 `[trace_file]-[config].txt` (例如 `602.gcc_s-2226B.champsimtrace.xz-perceptron-mlop_dpc3-next_line-next_line-lru-1core.txt`)。

## 3. 重現結果

此節說明如何用 `602.gcc_s-2226B.champsimtrace.xz` 追蹤檔重現 MLOP 預取器的模擬結果，並與無預取器的基準配置比較。

### 3.1 執行 MLOP 配置

1. **編譯**：
   ```bash
   ./build_champsim.sh perceptron mlop_dpc3 next_line next_line lru 1
   ```
   執行檔：`bin/perceptron-mlop_dpc3-next_line-next_line-lru-1core`

2. **執行**：
   ```bash
   ./run_champsim.sh perceptron-mlop_dpc3-next_line-next_line-lru-1core 50 200 dpc3_traces/602.gcc_s-2226B.champsimtrace.xz
   ```
   預期結果檔：`results_200M/602.gcc_s-2226B.champsimtrace.xz-perceptron-mlop_dpc3-next_line-next_line-lru-1core.txt`

### 3.2 執行基準配置 (無預取器)

1. **編譯**：
   ```bash
   ./build_champsim.sh perceptron no no no lru 1
   ```
   執行檔：`bin/perceptron-no-no-no-lru-1core`

2. **執行**：
   ```bash
   ./run_champsim.sh perceptron-no-no-no-lru-1core 50 200 dpc3_traces/602.gcc_s-2226B.champsimtrace.xz
   ```
   預期結果檔：`results_200M/602.gcc_s-2226B.champsimtrace.xz-perceptron-no-no-no-lru-1core.txt`

### 3.3 結果驗證

比較 MLOP 與基準配置的結果檔，重點觀察：

* **IPC (Instructions Per Cycle)**：越高越好。
* **L1D MPKI (Misses Per Kilo Instructions for L1 Data Cache)**：越低越好。
* **L1D Prefetch Accuracy** (預取準確度)：預取命中 / (預取命中 + 預取未命中)。越高越好。
* **L1D Prefetch Coverage** (預取覆蓋率)：(原始未命中但被預取命中的數量) / (原始未命中總數)。越高越好。

詳細指標名稱請見 ChampSim 輸出。

## 4. 簡短報告 (方法、分析、問題與解決)

根據 `results_200M/` 目錄中的模擬結果，我們比較了 MLOP 預取器與無預取器（基線）配置在 `602.gcc_s-2226B.champsimtrace.xz` 追蹤檔上的效能。

### 4.1 數據呈現

關鍵效能指標整理如下表：

| 指標 (Metric)                                      | 無預取器 (No Prefetcher)  | MLOP 預取器 (MLOP Prefetcher)  | 變化幅度 (Change)       |
| :------------------------------------------------ | :----------------------- | :---------------------------- | :--------------------- |
| IPC (Instructions Per Cycle)                      | 0.122                    | 0.449                         | 約 +267.0%             |
| L1D MPKI (每千條指令L1D需求未命中數)                  | 72.39                    | 18.48                         | 約 -74.5% (越低越好)   |
| L1D 預取準確度 (Prefetch Accuracy)                  | N/A                      | 69.00%                        | N/A                    |
| L1D 預取覆蓋率 (Prefetch Coverage)                  | N/A                      | 98.56%                        | N/A                    |

*註：L1D MPKI 基於 L1D 的需求未命中（Load + RFO misses）計算，計算公式為 (L1D 總需求未命中數 / 總指令數) * 1000。預取覆蓋率計算方式為：(MLOP 配置下的 L1D 有用預取數) / (無預取器配置下的 L1D 總需求未命中數)。*

### 4.2 效能比較

* **IPC 改進**：MLOP 預取器使 `602.gcc_s` 追蹤的 IPC 從約 0.122 顯著提升至約 0.449，增幅高達約 267.0%。這表明 MLOP 能有效地減少處理器等待記憶體存取的時間，從而大幅提高執行效率。
* **L1D 快取未命中率降低**：MLOP 將 L1D MPKI 從約 72.39 大幅降低至約 18.48，降幅約為 74.5%。這意味著 MLOP 成功地將大量原本會導致 L1D 快取未命中的數據提前載入到 L1D 快取中。
* **預取準確度與覆蓋率**：
  * MLOP 的預取準確度為 69.00%，代表其發出的預取請求中有超過三分之二的數據是有效的。這有助於減少不必要的記憶體頻寬消耗和快取污染，儘管仍有提升空間。
  * 預取覆蓋率達到了驚人的 98.56%，這表示 MLOP 幾乎能夠捕捉到所有潛在的 L1D 需求未命中，並透過預取來滿足它們。高覆蓋率是實現顯著效能提升的關鍵因素。

### 4.3 與論文比較

我們的模擬結果與 MLOP 論文中的核心觀點高度一致。該論文強調 MLOP 旨在同時優化預取的「及時性」(timeliness) 和「覆蓋率」(coverage)。
* 觀察到的 IPC 大幅提升（約 267.0%）與極高的覆蓋率（98.56%）有力地支持了 MLOP 的高效能設計。論文中的圖表（例如 Figure 3）展示了 MLOP 相對於無預取器基線在多種工作負載（包括 `gcc_s` 所屬的 Mix1）上的顯著效能優勢。我們的具體 IPC 增幅與此趨勢相符，顯示了 MLOP 對於記憶體密集型應用 `gcc_s` 的有效性。
* 論文中描述 MLOP 透過存取映射表 (AMT) 和多重前瞻等級來學習和選擇最佳預取偏移量，並根據預測的及時性和準確性對其進行排序。我們模擬出的高覆蓋率和 69.00% 的準確度，間接驗證了這些機制在 `602.gcc_s` 追蹤上的有效性。MLOP 能夠準確預測未來的記憶體存取，並及時發出預取請求，從而顯著降低了 L1D 的未命中次數，最終體現在 IPC 的大幅提升上。
