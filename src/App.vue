<script setup lang="ts">

/**
 * Composition API とユーティリティの取り込み
 */
import { ref, computed } from 'vue'
import { faker } from '@faker-js/faker'   // ダミーデータ生成
import yaml from 'js-yaml'                 // OpenAPI(YAML) → オブジェクト変換

const base: string = import.meta.env.BASE_URL; // 例: '/api-mock-demo/'

// ---- 型定義（セレクトの候補） ------------------------------------------
type HttpMethod = 'GET'|'POST'|'PUT'|'DELETE'

// ---- 画面入力のリアクティブ状態 -----------------------------------------
const endpoint = ref('/users')            // 入力：エンドポイント（例 /users）
const method   = ref<HttpMethod>('GET')   // 入力：HTTPメソッド
const count    = ref(5)                   // 入力：生成件数
// 入力：レスポンスの“簡易スキーマ”（キー: タイプ指定）
const schemaText = ref(`{
  "id": "uuid",
  "name": "string",
  "email": "email",
  "age": "number",
  "joined_at": "date:YYYY-MM-DD"
}`)

// 出力・状態
const result  = ref<any[]|null>(null)     // 生成結果（配列）を入れる
const error   = ref('')                   // 画面下に出すエラーメッセージ
const working = ref(false)                // 将来ローディング制御用（今は未使用）

// ---- 表示用の整形文字列 -----------------------------------------------
const prettyResult = computed(() =>
  result.value ? JSON.stringify(result.value, null, 2) : ''
)

// 生成した配列から MSW のハンドラ（rest.<method>(path, ...)）を文字列生成
const mswText = computed(() => {
  if (!result.value) return `// 生成後にここに出力されます`
  const data = JSON.stringify(result.value, null, 2)
  const ep = endpoint.value || '/'
  const m  = method.value.toLowerCase()
  return `import { rest } from 'msw'

export const handlers = [
  rest.${m}('${ep}', (req, res, ctx) => {
    return res(ctx.json(${data}))
  })
]`
})

/**
 * スキーマ文字列(JSON) → オブジェクトに変換
 * 入力例:
 *  { "id":"uuid", "email":"email" }
 */
function parseSchema(txt: string): Record<string,string> {
  const obj = JSON.parse(txt)
  if (!obj || typeof obj !== 'object') throw new Error('スキーマはオブジェクトで指定してください。')
  return obj as Record<string,string>
}

/**
 * スキーマの型指定（"uuid" / "string" / "date:YYYY-MM-DD" など）から
 * faker を使って1値を生成
 */
function genValue(typeSpec: string) {
  if (typeSpec.startsWith('date:')) {
    // 日付フォーマット指定（YYYY-MM-DD など）
    const fmt = typeSpec.slice(5)
    const dt = faker.date.between({ from: '2018-01-01', to: '2025-12-31' })
    const yyyy = dt.getFullYear(), mm = String(dt.getMonth()+1).padStart(2,'0'), dd = String(dt.getDate()).padStart(2,'0')
    if (fmt === 'YYYY-MM-DD') return `${yyyy}-${mm}-${dd}`
    if (fmt === 'YYYY/MM/DD') return `${yyyy}/${mm}/${dd}`
    return dt.toISOString()
  }
  // 基本型
  switch (typeSpec) {
    case 'uuid':   return faker.string.uuid()
    case 'string': return faker.person.fullName()
    case 'email':  return faker.internet.email().toLowerCase()
    case 'number': return faker.number.int({ min: 0, max: 9999 })
    default:       return null                // 未対応指定は null を返す
  }
}

/**
 * 1レコード（=1オブジェクト）をスキーマから生成
 */
function buildRecord(schema: Record<string,string>) {
  const rec: Record<string, any> = {}
  for (const [k, t] of Object.entries(schema)) rec[k] = genValue(t)
  return rec
}

/**
 * 「モック生成」ボタン：入力スキーマ & 件数から配列を作って result に格納
 */
async function onGenerate() {
  error.value = ''; working.value = true
  try {
    const schema = parseSchema(schemaText.value)
    // 件数は 1〜10000 の範囲に丸める
    const n = Math.max(1, Math.min(10000, Number(count.value) || 1))
    result.value = Array.from({ length: n }, () => buildRecord(schema))
  } catch (e: any) {
    error.value = e.message ?? String(e)
    result.value = null
  } finally { working.value = false }
}

/**
 * テキストをファイルとしてダウンロードさせるユーティリティ
 */
function download(name: string, text: string, mime='text/plain') {
  const blob = new Blob([text], { type: mime })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url; a.download = name; a.click()
  URL.revokeObjectURL(url)
}

/**
 * 生成した配列を JSON ファイルとして保存
 */
function onDownloadJson() {
  if (!result.value) return
  const safe = (endpoint.value || 'mock').replace(/[^\w\-]+/g, '_')
  download(
    `${method.value}_${safe}.json`,
    JSON.stringify(result.value, null, 2),
    'application/json'
  )
}

/**
 * 生成した MSW ハンドラ文字列をクリップボードへ
 */
function copyMsw() {
  if (!navigator.clipboard) return alert('Clipboard APIが使えません')
  navigator.clipboard.writeText(mswText.value).then(() => alert('コピーしました'))
}

/**
 * 生成した MSW ハンドラ文字列を .ts でダウンロード
 */
function downloadMsw() {
  if (!result.value) return
  const safe = (endpoint.value || 'mock').replace(/[^\w\-]+/g, '_')
  download(`msw_${method.value}_${safe}.ts`, mswText.value)
}

/* ===================== OpenAPI 読み込み関連 ===================== */

/**
 * OpenAPI の型 → 本ツールの簡易型表現 へマッピング
 *   - t: OpenAPIのtype, f: format
 */
function mapSchemaType(t?: string, f?: string): string {
  if (f === 'uuid')                 return 'uuid'
  if (f === 'email')                return 'email'
  if (f === 'date' || f === 'date-time') return 'date:YYYY-MM-DD'
  if (t === 'string')               return 'string'
  if (t === 'integer' || t === 'number') return 'number'
  return 'string'
}

/**
 * OpenAPIドキュメント（オブジェクト）から
 * 最初の path / method / 200 / application/json の schema を拾い、
 * { key: 簡易型 } の形に変換して返す（MVPの簡易対応）
 */
function openApiToSchema(doc: any): Record<string,string> {
  if (!doc?.paths) throw new Error('OpenAPI: paths が見つかりません')
  const path = Object.keys(doc.paths)[0]
  const method = Object.keys(doc.paths[path])[0]
  const resp = doc.paths[path][method]?.responses?.['200']
  const schema = resp?.content?.['application/json']?.schema
  if (!schema) throw new Error('application/json の schema が見つかりません')

  // properties を走査して簡易型へ
  const convertProps = (props: any): Record<string,string> => {
    const r: Record<string,string> = {}
    for (const [k, v] of Object.entries<any>(props || {})) {
      r[k] = mapSchemaType(v.type, v.format)
    }
    return r
  }

  // array の items.properties / object の properties に対応
  if (schema.type === 'array'  && schema.items?.properties) return convertProps(schema.items.properties)
  if (schema.type === 'object' && schema.properties)        return convertProps(schema.properties)
  throw new Error('未対応のOpenAPI schema（MVP簡易対応）')
}

/**
 * 入力ファイル(YAML / JSON)を読み込んで openApiToSchema → schemaText に反映
 */
function onOpenApiUpload(e: Event) {
  const input = e.target as HTMLInputElement
  const file = input.files?.[0]; if (!file) return
  const reader = new FileReader()
  reader.onload = () => {
    try {
      const text = String(reader.result)
      // 拡張子で YAML/JSON を判定
      const doc = /\.(ya?ml)$/i.test(file.name) ? yaml.load(text) : JSON.parse(text)
      const schema = openApiToSchema(doc)
      schemaText.value = JSON.stringify(schema, null, 2)
    } catch (err: any) {
      error.value = 'OpenAPI変換失敗: ' + (err.message ?? String(err))
    }
  }
  reader.readAsText(file)
}
</script>

<template>
  <!-- レイアウト：2カラム（左：入力、右：プレビュー） -->
  <main class="max-w-6xl mx-auto p-4 space-y-4">
    <h1 class="text-2xl font-bold">API Mock Generator</h1>

<script setup lang="ts">
// 既存の import/状態の下あたりに1行だけ追加
const base = import.meta.env.BASE_URL; // 例: '/api-mock-demo/'
</script>

<template>
  <main class="max-w-6xl mx-auto p-4 space-y-4">
    <!-- ここを丸ごと差し替え -->
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-bold">API Mock Generator</h1>

      <nav class="flex gap-2">
        <a :href="base + 'docs/'"
           target="_blank" rel="noopener"
           class="px-3 py-1.5 rounded bg-blue-500 text-white hover:bg-blue-600">
          API Docs (Swagger)
        </a>
        <a :href="base + 'docs/redoc.html'"
           target="_blank" rel="noopener"
           class="px-3 py-1.5 rounded bg-slate-600 text-white hover:bg-slate-700">
          API Docs (Redoc)
        </a>
      </nav>
    </div>

    <!-- 以下、あなたの既存のUI -->
    <!-- … -->
  </main>
</template>


    <p class="text-sm text-gray-600">モック生成 / MSW出力 / OpenAPI→スキーマ</p>

    <div class="grid md:grid-cols-2 gap-4">
      <!-- 左カラム：入力フォーム群 -->
      <div class="space-y-3">
        <label class="block font-medium">エンドポイント</label>
        <input v-model="endpoint" class="mt-1 w-full border rounded px-3 py-2" placeholder="/users" />

        <label class="block font-medium">HTTPメソッド</label>
        <select v-model="method" class="mt-1 w-full border rounded px-3 py-2">
          <option>GET</option><option>POST</option><option>PUT</option><option>DELETE</option>
        </select>

        <label class="block font-medium">件数</label>
        <input v-model.number="count" type="number" min="1" max="10000" class="mt-1 w-full border rounded px-3 py-2" />

        <label class="block font-medium">レスポンススキーマ（JSON）</label>
        <textarea
          v-model="schemaText"
          rows="12"
          class="mt-1 w-full border rounded px-3 py-2 font-mono text-sm"
        />

        <div class="flex gap-2">
          <!-- 生成 / JSON 保存 -->
          <button @click="onGenerate" class="px-4 py-2 rounded bg-blue-600 text-white hover:bg-blue-700">モック生成</button>
          <button @click="onDownloadJson" class="px-4 py-2 rounded bg-gray-800 text-white hover:bg-gray-900" :disabled="!result">JSONダウンロード</button>
        </div>

        <!-- OpenAPI 読み込み -->
        <div class="mt-4 space-y-2">
          <label class="block font-medium">OpenAPI (YAML/JSON) 読み込み</label>
          <input type="file" @change="onOpenApiUpload" accept=".yaml,.yml,.json" />
          <p class="text-xs text-gray-500">最初の path/method/200 の schema を読み取り、上のスキーマ欄に変換（MVP）。</p>
        </div>

        <!-- エラー表示 -->
        <p v-if="error" class="text-sm text-red-600 whitespace-pre-wrap">{{ error }}</p>
      </div>

      <!-- 右カラム：結果とMSWコード -->
      <div class="space-y-3">
        <div>
          <label class="block font-medium">生成結果（プレビュー）</label>
          <pre class="mt-1 w-full border rounded p-3 bg-white overflow-auto text-sm" style="max-height: 300px;">
{{ prettyResult }}</pre>
        </div>

        <div>
          <div class="flex items-center justify-between">
            <label class="block font-medium">MSWハンドラ出力</label>
            <div class="flex gap-2">
              <button class="px-3 py-1.5 rounded bg-slate-600 text-white text-sm" @click="copyMsw">コピー</button>
              <button class="px-3 py-1.5 rounded bg-slate-800 text-white text-sm" @click="downloadMsw" :disabled="!result">ダウンロード</button>
            </div>
          </div>
          <pre class="mt-1 w-full border rounded p-3 bg-white overflow-auto text-sm" style="max-height: 260px;">
{{ mswText }}</pre>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
/* いまはスタイル最小限。Tailwindベースなので必要に応じて追加してOK */
</style>
