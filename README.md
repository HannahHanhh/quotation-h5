# quotation-h5

一个纯前端的 H5 报价单生成器。在手机或电脑浏览器里填表（产品图片、名称、单价、单位、数量、备注），一键生成 PDF 下载到本地，文件名形如 `南山马总12月5号报价单.pdf`。

## 本地开发

```bash
npm install
npm run dev
```

## 构建

```bash
npm run build
```

## 部署

推送到 `main` 分支后，GitHub Actions 会自动构建并发布到 GitHub Pages。

## 技术栈

- Vue 3 + Vite
- Vant 4（移动端 UI）
- jsPDF + html2canvas（PDF 生成）
