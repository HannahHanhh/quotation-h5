<script setup>
import { ref, computed, reactive } from 'vue'
import { showToast, showLoadingToast, closeToast } from 'vant'
import html2canvas from 'html2canvas'
import { jsPDF } from 'jspdf'

const meta = reactive({
  title: '报价单',
  customer: '',
  date: formatToday(),
  note: ''
})

const items = ref([createItem()])
const previewRef = ref(null)

function createItem() {
  return {
    id: Date.now() + Math.random(),
    image: '',
    name: '',
    price: '',
    unit: '个',
    qty: '',
    note: ''
  }
}

function formatToday() {
  const d = new Date()
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
}

function addItem() {
  items.value.push(createItem())
}

function removeItem(i) {
  if (items.value.length === 1) {
    items.value[0] = createItem()
    return
  }
  items.value.splice(i, 1)
}

function onImageUpload(item, file) {
  const f = Array.isArray(file) ? file[0].file : file.file
  if (!f) return
  const reader = new FileReader()
  reader.onload = (e) => {
    item.image = e.target.result
  }
  reader.readAsDataURL(f)
}

function amount(it) {
  const p = parseFloat(it.price) || 0
  const q = parseFloat(it.qty) || 0
  return +(p * q).toFixed(2)
}

const total = computed(() =>
  items.value.reduce((sum, it) => sum + amount(it), 0).toFixed(2)
)

function pdfFileName() {
  const d = meta.date ? new Date(meta.date) : new Date()
  const m = d.getMonth() + 1
  const day = d.getDate()
  const name = meta.customer.trim() || '客户'
  return `${name}${m}月${day}号报价单.pdf`
}

async function generatePdf() {
  if (!items.value.some((it) => it.name.trim())) {
    showToast('请至少填写一项产品名称')
    return
  }
  showLoadingToast({ message: '生成中...', forbidClick: true, duration: 0 })
  try {
    await nextTickPaint()
    const el = previewRef.value
    const canvas = await html2canvas(el, {
      scale: 2,
      useCORS: true,
      backgroundColor: '#ffffff'
    })
    const imgData = canvas.toDataURL('image/jpeg', 0.92)

    const pdf = new jsPDF({ unit: 'mm', format: 'a4', orientation: 'portrait' })
    const pageW = pdf.internal.pageSize.getWidth()
    const pageH = pdf.internal.pageSize.getHeight()
    const margin = 8
    const imgW = pageW - margin * 2
    const imgH = (canvas.height * imgW) / canvas.width

    if (imgH <= pageH - margin * 2) {
      pdf.addImage(imgData, 'JPEG', margin, margin, imgW, imgH)
    } else {
      // 分页：按页切片
      const pageContentH = pageH - margin * 2
      const pxPerMm = canvas.width / imgW
      const pageContentPx = pageContentH * pxPerMm
      let rendered = 0
      while (rendered < canvas.height) {
        const sliceH = Math.min(pageContentPx, canvas.height - rendered)
        const slice = document.createElement('canvas')
        slice.width = canvas.width
        slice.height = sliceH
        const ctx = slice.getContext('2d')
        ctx.drawImage(canvas, 0, rendered, canvas.width, sliceH, 0, 0, canvas.width, sliceH)
        const sliceData = slice.toDataURL('image/jpeg', 0.92)
        const sliceMm = sliceH / pxPerMm
        if (rendered > 0) pdf.addPage()
        pdf.addImage(sliceData, 'JPEG', margin, margin, imgW, sliceMm)
        rendered += sliceH
      }
    }

    pdf.save(pdfFileName())
    closeToast()
    showToast('已保存到下载目录')
  } catch (e) {
    closeToast()
    showToast('生成失败：' + (e.message || e))
  }
}

function nextTickPaint() {
  return new Promise((r) => requestAnimationFrame(() => requestAnimationFrame(r)))
}
</script>

<template>
  <div class="page">
    <van-nav-bar title="报价单生成" />

    <van-cell-group inset title="基本信息">
      <van-field v-model="meta.title" label="标题" placeholder="例如：报价单" />
      <van-field v-model="meta.customer" label="报价单名称" placeholder="例如：南山马总" />
      <van-field v-model="meta.date" label="日期" type="date" placeholder="选择日期" />
      <van-field v-model="meta.note" label="整体备注" rows="2" autosize type="textarea" placeholder="选填" />
    </van-cell-group>

    <van-cell-group v-for="(it, i) in items" :key="it.id" inset :title="`产品 ${i + 1}`" class="item-group">
      <template #title>
        <div class="item-title">
          <span>产品 {{ i + 1 }}</span>
          <van-button size="mini" type="danger" plain @click="removeItem(i)">删除</van-button>
        </div>
      </template>
      <van-field label="图片">
        <template #input>
          <van-uploader
            :max-count="1"
            :after-read="(f) => onImageUpload(it, f)"
            :preview-image="true"
            :model-value="it.image ? [{ url: it.image }] : []"
            @delete="() => (it.image = '')"
          />
        </template>
      </van-field>
      <van-field v-model="it.name" label="产品名称" placeholder="必填" />
      <van-field v-model="it.price" label="单价" type="number" placeholder="元">
        <template #extra>元</template>
      </van-field>
      <van-field v-model="it.unit" label="单位" placeholder="个/箱/件..." />
      <van-field v-model="it.qty" label="数量" type="number" />
      <van-field label="金额">
        <template #input>
          <span class="amt">¥ {{ amount(it).toFixed(2) }}</span>
        </template>
      </van-field>
      <van-field v-model="it.note" label="备注" placeholder="选填" />
    </van-cell-group>

    <div class="actions">
      <van-button block type="default" @click="addItem">+ 添加产品</van-button>
      <van-button block type="primary" @click="generatePdf" style="margin-top: 12px">
        生成 PDF 报价单
      </van-button>
    </div>

    <!-- 离屏渲染的预览，用于 html2canvas -->
    <div class="preview-wrap">
      <div ref="previewRef" class="preview">
        <h1 class="pv-title">{{ meta.title || '报价单' }}</h1>
        <div class="pv-meta">
          <div><span>名称：</span>{{ meta.customer || '—' }}</div>
          <div><span>日期：</span>{{ meta.date }}</div>
        </div>
        <table class="pv-table">
          <colgroup>
            <col style="width: 44px" />
            <col />
            <col style="width: 90px" />
            <col style="width: 70px" />
            <col style="width: 50px" />
            <col style="width: 50px" />
            <col style="width: 80px" />
            <col />
          </colgroup>
          <thead>
            <tr>
              <th>序号</th>
              <th>产品名称</th>
              <th>图片</th>
              <th>单价(元)</th>
              <th>单位</th>
              <th>数量</th>
              <th>金额(元)</th>
              <th>备注</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(it, i) in items" :key="it.id">
              <td>{{ i + 1 }}</td>
              <td>{{ it.name || '—' }}</td>
              <td>
                <img v-if="it.image" :src="it.image" class="pv-img" />
                <span v-else class="muted">—</span>
              </td>
              <td>{{ it.price ? Number(it.price).toFixed(2) : '—' }}</td>
              <td>{{ it.unit || '—' }}</td>
              <td>{{ it.qty || '—' }}</td>
              <td>{{ amount(it).toFixed(2) }}</td>
              <td>{{ it.note || '' }}</td>
            </tr>
            <tr class="total-row">
              <td colspan="6" style="text-align: right">合计(元)</td>
              <td colspan="2">¥ {{ total }}</td>
            </tr>
          </tbody>
        </table>
        <div v-if="meta.note" class="pv-note"><b>备注：</b>{{ meta.note }}</div>
      </div>
    </div>
  </div>
</template>

<style>
.page {
  padding: 12px 0 80px;
}
.item-group {
  margin-top: 12px;
}
.item-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}
.actions {
  padding: 16px;
}
.amt {
  color: #ee0a24;
  font-weight: 600;
}

/* 预览样式：放在视图外，用于 html2canvas 截图 */
.preview-wrap {
  position: fixed;
  left: -99999px;
  top: 0;
  pointer-events: none;
}
.preview {
  width: 794px; /* A4 @ 96dpi */
  padding: 32px 28px;
  background: #fff;
  color: #000;
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Microsoft YaHei', sans-serif;
  box-sizing: border-box;
}
.pv-title {
  text-align: center;
  margin: 0 0 16px;
  font-size: 26px;
  letter-spacing: 4px;
}
.pv-meta {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  margin-bottom: 10px;
}
.pv-meta span {
  color: #666;
}
.pv-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}
.pv-table th,
.pv-table td {
  border: 1px solid #888;
  padding: 6px 4px;
  text-align: center;
  vertical-align: middle;
  word-break: break-all;
}
.pv-table th {
  background: #f0f0f0;
  font-weight: 600;
}
.pv-img {
  max-width: 80px;
  max-height: 60px;
  object-fit: contain;
  display: block;
  margin: 0 auto;
}
.muted {
  color: #aaa;
}
.total-row td {
  background: #fafafa;
  font-weight: 600;
  color: #d00;
}
.pv-note {
  margin-top: 14px;
  font-size: 13px;
  line-height: 1.6;
}
</style>
