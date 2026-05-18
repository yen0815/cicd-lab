# CI Pipeline 報告

## 1. CI Pipeline 說明

已新增檔案：`.github/workflows/ci_313552054.yaml`

### 主要內容

```yaml
name: CI 313552054

on:
  push:
    branches:
      - '**'

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: TypeScript typecheck
        run: npm run typecheck

      - name: Prettier format check
        run: npm run format:check

      - name: Run tests and collect JUnit report
        run: |
          mkdir -p reports
          npm test -- --reporter=default --reporter=junit --outputFile=reports/vitest-junit.xml

      - name: Upload test report artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: vitest-report
          path: reports/vitest-junit.xml
          if-no-files-found: error

      - name: Publish Vitest summary to workflow page
        if: always()
        run: |
          python3 - <<'PY'
          import os
          import xml.etree.ElementTree as ET

          report_path = 'reports/vitest-junit.xml'
          if not os.path.exists(report_path):
              print('No JUnit report found')
              raise SystemExit(0)

          tree = ET.parse(report_path)
          root = tree.getroot()
          suite = root.find('.//testsuite') or root

          tests = int(suite.attrib.get('tests', 0))
          failures = int(suite.attrib.get('failures', 0))
          errors = int(suite.attrib.get('errors', 0))
          skipped = int(suite.attrib.get('skipped', suite.attrib.get('disabled', 0)))

          summary = (
              f'## Vitest Results\n'
              f'- Tests: {tests}\n'
              f'- Failures: {failures}\n'
              f'- Errors: {errors}\n'
              f'- Skipped: {skipped}\n'
          )

          print(summary)
          with open(os.environ['GITHUB_STEP_SUMMARY'], 'a') as fh:
              fh.write(summary)
          PY
```

### Pipeline 設計說明

- 觸發條件為 `push` 事件，任何 branch 推送時都會自動執行。
- 使用 `actions/checkout@v4`、`actions/setup-node@v4`、`actions/upload-artifact@v4` 等現成 GitHub Actions。
- 以專案現有 `package.json` scripts 為基礎：
  - `npm run typecheck`：TypeScript 靜態檢查。
  - `npm run format:check`：Prettier 格式檢查。
  - `npm test`：執行 Vitest 測試。
- 測試執行後產生 JUnit XML 報告，並上傳為 artifact，方便後續檢視與整合。
- 最後一步將測試結果寫入 `GITHUB_STEP_SUMMARY`，讓 GitHub Actions 結果頁面可直接看到測試統計摘要。
- 只要任一檢查失敗，整個 job 會以非零狀態結束，GitHub Actions 會顯示失敗。

## 2. CI 執行結果

### 本地驗證命令

```bash
npm run typecheck
npm run format:check
mkdir -p reports
npm test -- --reporter=default --reporter=junit --outputFile=reports/vitest-junit.xml
```

### 驗證結果摘要

- TypeScript typecheck：成功
- Prettier 檢查：成功
- 測試執行：成功，生成報告 `reports/vitest-junit.xml`

### 截圖說明

- 成功執行截圖：請在 GitHub Actions 執行成功後擷取 `ci_313552054` workflow 的結果頁面。
- 圖片可放置於 `screenshots/ci-success.png`。

## 3. 失敗案例說明

### 範例錯誤：TypeScript 型別錯誤

可故意在 `src/app.ts` 或 `src/server.ts` 中新增錯誤型別，例如：

```ts
const port: string = 3000;
```

這會導致 `npm run typecheck` 失敗，且 pipeline 在 `TypeScript typecheck` 步驟立即停止。

### 範例錯誤：Prettier 格式錯誤

可故意新增未格式化的程式片段，例如：

```ts
const x = 1;
```

這會導致 `npm run format:check` 失敗，pipeline 在 `Prettier format check` 步驟失敗。

### 範例錯誤：測試失敗

可修改 `test/app.test.ts` 中的斷言，讓測試失敗：

```ts
expect(response.statusCode).toBe(404);
```

這會導致 `npm test` 失敗，並且 `Upload test report artifact` 仍會執行（因為使用 `if: always()`），但整個 job 仍會標記失敗。

### 錯誤修正方式

- TypeScript 錯誤：修正型別不一致的程式碼，讓 `tsc --noEmit` 通過。
- Prettier 錯誤：執行 `npm run format` 或手動修正格式，讓 `prettier --check .` 通過。
- 測試失敗：修正測試斷言或對應功能程式碼，讓 `vitest run` 回傳成功。

### 失敗截圖說明

- 失敗截圖：請在 GitHub Actions failure run 中擷取 `ci_313552054` workflow 的結果頁面。
- 圖片可放置於 `screenshots/ci-failed.png`。

## 4. 小結

此報告包含

- CI workflow 主要內容
- pipeline 設計說明
- 本地驗證結果摘要
- 失敗案例與修正方式

若要補齊作業附件，可將 GitHub Actions 的成功與失敗截圖放在 `screenshots/` 目錄，並更新本報告中的圖片連結。
