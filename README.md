

https://github.com/user-attachments/assets/4dedbcd7-895f-4ac4-a3a5-baa4545acb41

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// ⚙️ การตั้งค่า — ปรับตามการใช้งาน
const CONFIG = {
  GOOGLE_SHEET_ID: '', // ใส่ ID Google Sheet ที่นี่
  SHEET_NAME: 'ข้อมูล',
  GOOGLE_API_KEY: '',
  SEARCH_ENGINE_ID: '',
  LEGAL_RULES: {
    MIN_AGE: 18,
    MAX_AGE: 100,
    MIN_REGISTRATION_AGE: 15
  }
} as const

interface RegisteredPerson {
  id: string
  fullName: string
  birthDate: string
  age: number
  registrationDate: string
  status: 'active' | 'suspended' | 'expired'
  idNumber: string
  isDataComplete: boolean
  source?: 'local' | 'google-sheet'
}

interface SearchResult {
  title: string
  link: string
  snippet: string
}

// 📐 คำนวณอายุ — ปรับปรุงความแม่นยำ
function calculateAge(birthDateStr: string): number {
  if (!birthDateStr?.trim()) return 0
  const birth = new Date(birthDateStr)
  if (isNaN(birth.getTime())) return 0
  const today = new Date()
  let age = today.getFullYear() - birth.getFullYear()
  const monthDiff = today.getMonth() - birth.getMonth()
  if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birth.getDate())) age--
  return Math.max(0, age)
}

// ⚖️ ตรวจสอบความถูกต้อง — เพิ่มตรวจสอบรูปแบบเลขบัตร
function checkLegalCompliance(person: RegisteredPerson) {
  const reasons: string[] = [], warnings: string[] = []

  // ตรวจสอบอายุ
  if (person.age < CONFIG.LEGAL_RULES.MIN_REGISTRATION_AGE) {
    reasons.push(`อายุต่ำกว่าเกณฑ์ขั้นต่ำ (ต้องมีอย่างน้อย ${CONFIG.LEGAL_RULES.MIN_REGISTRATION_AGE} ปี)`)
  } else if (person.age < CONFIG.LEGAL_RULES.MIN_AGE) {
    warnings.push(`อายุ ${person.age} ปี — ต้องมีผู้ปกครองลงนามรับรองตามกฎหมาย`)
  }
  if (person.age > CONFIG.LEGAL_RULES.MAX_AGE) {
    reasons.push(`อายุเกินขอบเขตที่ยอมรับ (ไม่เกิน ${CONFIG.LEGAL_RULES.MAX_AGE} ปี)`)
  }

  // ตรวจสอบสถานะ
  if (person.status !== 'active') {
    reasons.push(`สถานะ: ${person.status === 'suspended' ? 'ถูกระงับ' : 'หมดอายุ'} — ไม่สามารถใช้งานได้`)
  }

  // ตรวจสอบความครบถ้วนข้อมูล
  if (!person.isDataComplete) {
    reasons.push('ข้อมูลไม่ครบถ้วนตามที่กฎหมายกำหนด')
  }

  // ตรวจสอบรูปแบบเลขบัตรประชาชนไทย (13 หลัก ไม่มีขีด)
  const idNumClean = person.idNumber.replace(/[-\s]/g, '')
  if (person.idNumber && (idNumClean.length !== 13 || !/^\d+$/.test(idNumClean))) {
    warnings.push('รูปแบบเลขบัตรประจำตัวไม่ถูกต้อง — ควรเป็น 13 หลัก')
  }

  return { compliant: reasons.length === 0, reasons, warnings }
}

// 📋 === เชื่อมต่อ Google Sheets ===
const isLoadingSheet = ref(false)
const sheetError = ref('')
const lastUpdated = ref('')
const allPersons = ref<RegisteredPerson[]>([])

async function loadFromGoogleSheet() {
  if (!CONFIG.GOOGLE_SHEET_ID) {
    sheetError.value = '⚠️ ยังไม่ได้ตั้งค่า Google Sheet — ใช้ข้อมูลตัวอย่างแทน'
    loadSampleData()
    return
  }

  isLoadingSheet.value = true
  sheetError.value = ''
  try {
    const csvUrl = `https://docs.google.com/spreadsheets/d/${CONFIG.GOOGLE_SHEET_ID}/gviz/tq?tqx=out:csv&sheet=${encodeURIComponent(CONFIG.SHEET_NAME)}`
    const controller = new AbortController()
    const timeout = setTimeout(() => controller.abort(), 10000) // หมดเวลา 10 วินาที

    const res = await fetch(csvUrl, { signal: controller.signal })
    clearTimeout(timeout)
    if (!res.ok) throw new Error(`HTTP ${res.status}: ตรวจสอบสิทธิ์เผยแพร่ Sheet`)

    const csvText = await res.text()
    if (!csvText.trim()) throw new Error('ไม่พบข้อมูลใน Sheet')

    // แปลง CSV — ปรับปรุงรองรับคอมม่าภายในข้อความ
    const rows: string[][] = []
    const lines = csvText.split('\n')
    for (const line of lines) {
      if (!line.trim()) continue
      const cells: string[] = []
      let current = '', inQuote = false
      for (const ch of line) {
        if (ch === '"') inQuote = !inQuote
        else if (ch === ',' && !inQuote) { cells.push(current.trim()); current = '' }
        else current += ch
      }
      cells.push(current.trim())
      if (cells.some(c => c)) rows.push(cells)
    }

    if (rows.length < 2) throw new Error('ไม่พบข้อมูล แค่หัวตาราง')

    const headers = rows[0].map(h => h.trim())
    const dataRows = rows.slice(1)

    allPersons.value = dataRows.map(row => {
      const get = (keys: string[]) => {
        for (const k of keys) {
          const idx = headers.findIndex(h => h.toLowerCase() === k.toLowerCase())
          if (idx !== -1 && row[idx] !== undefined) return row[idx].trim()
        }
        return ''
      }

      const birthDate = get(['วันเกิด', 'birthDate', 'วันเดือนปีเกิด'])
      const age = calculateAge(birthDate)
      const statusVal = get(['สถานะ', 'status']).toLowerCase()
      const status: 'active' | 'suspended' | 'expired' =
        statusVal.includes('ระงับ') || statusVal.includes('suspend') ? 'suspended' :
        statusVal.includes('หมดอายุ') || statusVal.includes('expire') ? 'expired' : 'active'

      const id = get(['รหัสทะเบียน', 'id', 'เลขทะเบียน'])
      const fullName = get(['ชื่อ', 'fullName', 'ชื่อ-นามสกุล'])

      return {
        id, fullName, birthDate, age, status,
        regi
