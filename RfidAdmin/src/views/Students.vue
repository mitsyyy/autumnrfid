<script setup lang="ts">
import { ref, defineAsyncComponent, computed, watch } from 'vue'
import type { Student } from '@/typescript/models'
import mockData from '@/mock/models.json'

const ConfirmationModal = defineAsyncComponent(() => import('@/components/ConfirmationModal.vue'))
const UnsavedChangesModal = defineAsyncComponent(() => import('@/components/UnsavedChangesModal.vue'))
const Sidebar = defineAsyncComponent(() => import("@/components/Sidebar.vue"))
const Searchbar = defineAsyncComponent(() => import("@/components/Searchbar.vue"))

const students = ref<Student[]>(mockData.students as Student[])
const courses = mockData.courses
const yearLevels = mockData.yearLevels
const statusOptions = mockData.statusOptions 
const deleteStatusOptions = mockData.deleteStatusOptions

const formData = ref({
  id: '',
  firstName: '',
  lastName: '',
  middleName: '',
  suffix: '',
  course: '',
  block: '',
  yearLevel: '',
  status: '',
  birthday: '',
  email: '',
  phone: ''
})

// STATE 
const isAddModalOpen = ref<boolean>(false)
const isEditModalOpen = ref<boolean>(false)
const selectedStudentId = ref<string | null>(null)
const searchQuery = ref<string>('')
const isConfirmationModalOpen = ref<boolean>(false)
const isUnsavedChangesModalOpen = ref<boolean>(false)
const modalToClose = ref<'add' | 'edit' | null>(null)
const activeFilters = ref<string[]>([])

// GET BLOCK OPTIONS BASED ON COURSE AND YEAR LEVEL
const getBlockOptions = (course: string, yearLevel: string) => {
  if (!course || !yearLevel) return []
  const year = yearLevel.charAt(0)
  const coursePrefix = course.replace('BS', '')
  const sections = ['A', 'B', 'C']
  return sections.map(section => `${coursePrefix}${year}1${section}`)
}

// BLOCK OPTIONS BASED ON FORM DATA
const blockOptions = computed(() => {
  const options = getBlockOptions(formData.value.course, formData.value.yearLevel)
  if (formData.value.block && !options.includes(formData.value.block)) {
    const blockPattern = /^[A-Z]{2}\d{2}[A-Z]$/
    if (blockPattern.test(formData.value.block)) {
      options.push(formData.value.block)
    }
  }
  return options
})

watch(() => formData.value.course, () => {
  formData.value.block = ''
})

watch(() => formData.value.yearLevel, () => {
  formData.value.block = ''
})


// OPEN ADD STUDENT MODAL WITH EMPTY FORM 
const openAddModal = () => {
  formData.value = {
    id: '',
    firstName: '',
    lastName: '',
    middleName: '',
    suffix: '',
    course: '',
    block: '',
    yearLevel: '',
    status: '',
    birthday: '',
    email: '',
    phone: ''
  }
  isAddModalOpen.value = true
}

// CLOSE ADD STUDENT MODAL 
const closeAddModal = () => {
  if (hasUnsavedChanges.value) {
    modalToClose.value = 'add'
    isUnsavedChangesModalOpen.value = true
  } else {
    isAddModalOpen.value = false
    formData.value = {
      id: '',
      firstName: '',
      lastName: '',
      middleName: '',
      suffix: '',
      course: '',
      block: '',
      yearLevel: '',
      status: '',
      birthday: '',
      email: '',
      phone: ''
    }
  }
}

// OPEN EDIT STUDENT MODAL 
const openEditModal = (studentId: string) => {
  const student = students.value.find(s => s.id === studentId)
  if (student) {
    formData.value = { ...student }
    selectedStudentId.value = studentId
    isEditModalOpen.value = true
  }
}

// CLOSES EDIT STUDENT MODAL 
const closeEditModal = () => {
  const originalStudent = students.value.find(s => s.id === selectedStudentId.value)
  if (!originalStudent) {
    isEditModalOpen.value = false
    selectedStudentId.value = null
    return
  }

  if (hasUnsavedChanges.value) {
    modalToClose.value = 'edit'
    isUnsavedChangesModalOpen.value = true
  } else {
    isEditModalOpen.value = false
    selectedStudentId.value = null
  }
}

// TRANSFORM INPUT TO CAMEL CASE
const toCamelCase = (str: string) => {
  return str
    .trim()
    .toLowerCase()
    .replace(/\w\S*/g, (word) => (
      word.charAt(0).toUpperCase() + word.substr(1).toLowerCase()
    ));
}

// VALIDATE NAME INPUT
const validateNameInput = (event: Event) => {
  const input = event.target as HTMLInputElement
  const cursorPosition = input.selectionStart ?? 0
  
  const originalValue = input.value
  const regex = /[^a-zA-Z\s.,-]/g
  const newValue = originalValue.replace(regex, '')
  
  if (newValue !== originalValue) {
    input.value = newValue
    input.dispatchEvent(new Event('input'))
    input.setSelectionRange(cursorPosition - 1, cursorPosition - 1)
  }
}

// VALIDATE PHONE INPUT
const validatePhoneInput = (event: Event) => {
  const input = event.target as HTMLInputElement
  const regex = /[^0-9+]/g
  input.value = input.value.replace(regex, '')
}

// VALIDATE BIRTH YEAR
const validateBirthYear = (birthday: string) => {
  const birthYear = new Date(birthday).getFullYear()
  return birthYear < 2010
}

// HANDLE ADD STUDENT SUBMISSION 
const handleSubmitAdd = (e: Event) => {
  e.preventDefault()
  
  const studentData = { ...formData.value }
  
  // BIRTHDAY VALIDATION
  if (!validateBirthYear(studentData.birthday)) {
    alert('Students born in 2010 or later are not eligible for admission.')
    return
  }
  
  studentData.id = studentData.id.toUpperCase()
  studentData.firstName = toCamelCase(studentData.firstName)
  studentData.lastName = toCamelCase(studentData.lastName)
  studentData.middleName = studentData.middleName ? toCamelCase(studentData.middleName) : ''
  studentData.suffix = studentData.suffix ? toCamelCase(studentData.suffix) : ''
  
  if (isStudentIdExists(studentData.id)) {
    alert('Student ID already exists. Please use a different ID.')
    return
  }
  
  students.value.push(studentData as Student)
  isAddModalOpen.value = false

  // RESET AFTER SAVING
  formData.value = {
    id: '',
    firstName: '',
    lastName: '',
    middleName: '',
    suffix: '',
    course: '',
    block: '',
    yearLevel: '',
    status: '',
    birthday: '',
    email: '',
    phone: ''
  }
}

// HANDLE EDIT STUDENT SUBMISSION 
const handleSubmitEdit = (e: Event) => {
  e.preventDefault()
  
  const studentData = { ...formData.value }
  
  // BIRTHDAY VALIDATION
  if (!validateBirthYear(studentData.birthday)) {
    alert('Students born in 2010 or later are not eligible for admission.')
    return
  }
  
  studentData.id = studentData.id.toUpperCase()
  studentData.firstName = toCamelCase(studentData.firstName)
  studentData.lastName = toCamelCase(studentData.lastName)
  studentData.middleName = studentData.middleName ? toCamelCase(studentData.middleName) : ''
  studentData.suffix = studentData.suffix ? toCamelCase(studentData.suffix) : ''
  
  if (isStudentIdExists(studentData.id, selectedStudentId.value)) {
    alert('Student ID already exists. Please use a different ID.')
    return
  }
  
  const index = students.value.findIndex(s => s.id === selectedStudentId.value)
  if (index !== -1) {
    students.value[index] = studentData as Student
    isEditModalOpen.value = false
    selectedStudentId.value = null

    // RESET AFTER SAVING
    formData.value = {
      id: '',
      firstName: '',
      lastName: '',
      middleName: '',
      suffix: '',
      course: '',
      block: '',
      yearLevel: '',
      status: '',
      birthday: '',
      email: '',
      phone: ''
    }
  }
}

// CONFIRMATION MODAL DATA 
const confirmationData = ref({
  title: '',
  itemName: '',
  itemInfo: null as Student | null
})

// SHOW DELETE CONFIRMATION MODAL
const showDeleteConfirmation = (title: string, itemName: string, itemInfo: Student) => {
  confirmationData.value = { 
    title, 
    itemName: `${itemName} (This will remove the student from this list and mark them as Dropped or Withdrawn)`, 
    itemInfo 
  }
  isConfirmationModalOpen.value = true
}

// HANDLE DELETE CONFIRMATION 
const handleConfirmDelete = (itemInfo: Student & { newStatus: string }) => {
  const index = students.value.findIndex(s => s.id === itemInfo.id)
  if (index !== -1) {
    if (itemInfo.newStatus && (itemInfo.newStatus === 'Dropped' || itemInfo.newStatus === 'Withdrawn')) {
      students.value[index].status = itemInfo.newStatus
    } else {
      students.value[index].status = 'Withdrawn'
    }
    students.value = students.value.filter(student => 
      student.id !== itemInfo.id || 
      (student.status !== 'Dropped' && student.status !== 'Withdrawn')
    )
  }
  isConfirmationModalOpen.value = false
}

// HANDLE SEARCH QUERY 
const handleSearch = (query: string) => {
  searchQuery.value = query || ''
}

// HANDLE FILTER CHANGES 
const handleFilterChange = (filters: string[]) => {
  activeFilters.value = filters || []
}

// FILTERED STUDENTS
const filteredStudents = computed(() => {
  return students.value.filter(student => {
    if (student.status === 'Dropped' || student.status === 'Withdrawn') {
      return false
    }
    
    const fullName = getFullName(student).toLowerCase()
    const matchesSearch = !searchQuery.value ||
      student.id.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      fullName.includes(searchQuery.value.toLowerCase()) ||
      student.course.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      student.yearLevel.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      student.status.toLowerCase().includes(searchQuery.value.toLowerCase())

    const matchesFilters = !activeFilters.value.length ||
      activeFilters.value.some(filter =>
        student.yearLevel.includes(filter) ||
        student.course.includes(filter) ||
        student.status.includes(filter))

    return matchesSearch && matchesFilters
  })
})

// CHECK IF STUDENT ID EXISTS 
const isStudentIdExists = (id: string, excludeCurrentId: string | null = null) => {
  return students.value.some(student => 
    student.id === id.toUpperCase() && 
    student.id !== (excludeCurrentId || '').toUpperCase()
  )
}

// GET FULL NAME OF STUDENT 
const getFullName = (student: Student) => {
  return `${student.firstName} ${student.lastName}`.trim()
}

// CHECK FOR UNSAVED CHANGES 
const hasUnsavedChanges = computed(() => {
  if (isAddModalOpen.value) {
    return Object.values(formData.value).some(value => value !== '')
  } else if (isEditModalOpen.value) {
    const originalStudent = students.value.find(s => s.id === selectedStudentId.value)
    if (!originalStudent) return false

    return Object.keys(formData.value).some(key => {
      const originalValue = String(originalStudent[key as keyof Student] || '').trim()
      const currentValue = String(formData.value[key as keyof typeof formData.value] || '').trim()
      
      if (!originalValue && !currentValue) return false
      
      return originalValue !== currentValue
    })
  }
  return false
})

// HANDLE UNSAVED CHANGES MODAL 
const handleUnsavedChanges = (confirm: boolean) => {
  if (confirm) {
    if (modalToClose.value === 'add') {
      isAddModalOpen.value = false
      formData.value = {
        id: '',
        firstName: '',
        lastName: '',
        middleName: '',
        suffix: '',
        course: '',
        block: '',
        yearLevel: '',
        status: '',
        birthday: '',
        email: '',
        phone: ''
      }
    } else if (modalToClose.value === 'edit') {
      isEditModalOpen.value = false
      selectedStudentId.value = null
    }
  }
  isUnsavedChangesModalOpen.value = false
  modalToClose.value = null
}
</script>

<template>
  <main>
    <div class="sidebar">
      <Sidebar />
    </div>

    <section>
      <div class="container">
        <div class="welcome-header">
          <h1>Manage Students</h1>
          <p>Add, edit, or remove student records</p>
        </div>

        <!-- SEARCH BAR SECTION -->
        <div class="students-controls">
          <div class="search-filters">
            <Searchbar :v-model="searchQuery" @update:search-query="handleSearch" @filter-change="handleFilterChange" />
            <div class="filter-buttons">
              <button class="add-student-btn" @click="openAddModal">
                <i class="fa-solid fa-plus"></i> Add New Student
              </button>
            </div>
          </div>
        </div>

        <!-- STUDENTS TABLE -->
        <div class="students-table">
          <table>
            <thead>
              <tr>
                <th>Student ID</th>
                <th>Name</th>
                <th>Course</th>
                <th>Year Level</th>
                <th>Status</th>
                <th>Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="student in filteredStudents" :key="student.id">
                <td>{{ student.id.toUpperCase() }}</td>
                <td>{{ getFullName(student) }}</td>
                <td>{{ student.course }}</td>
                <td>{{ student.yearLevel }}</td>
                <td>
                  <span :class="['status-badge', `status-${student.status.toLowerCase()}`]">
                    {{ student.status }}
                  </span>
                </td>
                <td class="action-buttons">
                  <button class="action-btn edit-btn" @click="openEditModal(student.id)">
                    <i class="fa-solid fa-pen-to-square"></i>
                  </button>
                  <button class="action-btn delete-btn" @click="showDeleteConfirmation(
                    'Delete Student',
                    `student with ID ${student.id}`,
                    student
                  )">
                    <i class="fa-solid fa-trash"></i>
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>


      <!-- ADD STUDENT MODAL -->
      <div class="modal" :class="{ active: isAddModalOpen }" @click="closeAddModal">
        <div class="modal-content" @click.stop>
          <div class="modal-header">
            <h2>Add New Student</h2>
            <button class="close-modal" @click="closeAddModal">&times;</button>
          </div>
          <form @submit="handleSubmitAdd" class="student-form">
            <div class="student-info-grid">
              <div class="info-group">
                <label>Student ID</label>
                <input type="text" v-model="formData.id" placeholder="Enter Student ID" required>
              </div>
              <div class="info-group">
                <label>First Name</label>
                <input 
                  type="text" 
                  v-model="formData.firstName" 
                  placeholder="Enter First Name" 
                  @input="validateNameInput"
                  required
                >
              </div>
              <div class="info-group">
                <label>Last Name</label>
                <input 
                  type="text" 
                  v-model="formData.lastName" 
                  placeholder="Enter Last Name" 
                  @input="validateNameInput"
                  required
                >
              </div>
              <div class="info-group">
                <label>Middle Name</label>
                <input 
                  type="text" 
                  v-model="formData.middleName" 
                  placeholder="Enter Middle Name" 
                  @input="validateNameInput"
                >
              </div>
              <div class="info-group">
                <label>Suffix</label>
                <input 
                  type="text" 
                  v-model="formData.suffix" 
                  placeholder="Enter Suffix (Optional)"
                  @input="validateNameInput"
                >
              </div>
              <div class="info-group">
                <label>Birthday</label>
                <input 
                  type="date" 
                  v-model="formData.birthday"
                  :max="'2009-12-31'"
                  required
                >
              </div>
              <div class="info-group">
                <label>Course</label>
                <select v-model="formData.course" required>
                  <option value="" disabled selected>Select Course</option>
                  <option v-for="course in courses" :key="course" :value="course">
                    {{ course }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Block</label>
                <select 
                  v-model="formData.block" 
                  required
                  :disabled="!formData.course || !formData.yearLevel"
                >
                  <option value="" disabled selected>
                    {{ !formData.course || !formData.yearLevel ? 
                      'Select Course and Year Level first' : 
                      'Select Block' 
                    }}
                  </option>
                  <option 
                    v-for="block in blockOptions" 
                    :key="block" 
                    :value="block"
                  >
                    {{ block }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Year Level</label>
                <select v-model="formData.yearLevel" required>
                  <option value="" disabled selected>Select Year Level</option>
                  <option v-for="year in yearLevels" :key="year" :value="year">
                    {{ year }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Status</label>
                <select v-model="formData.status" required>
                  <option value="" disabled selected>Select Status</option>
                  <option v-for="status in statusOptions" :key="status" :value="status">
                    {{ status }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Email</label>
                <input 
                  type="email" 
                  v-model="formData.email" 
                  placeholder="Enter Student Email" 
                  required
                >
              </div>
              <div class="info-group">
                <label>Phone</label>
                <input 
                  type="tel" 
                  v-model="formData.phone" 
                  placeholder="Enter Student Phone" 
                  @input="validatePhoneInput"
                  pattern="[0-9+]+"
                  required
                >
              </div>
            </div>
            <div class="form-actions">
              <button type="button" class="cancel-btn" @click="closeAddModal">Cancel</button>
              <button type="submit" class="submit-btn">Add Student</button>
            </div>
          </form>
        </div>
      </div>


      <!-- EDIT STUDENT MODAL -->
      <div class="modal" :class="{ active: isEditModalOpen }" @click="closeEditModal">
        <div class="modal-content" @click.stop>
          <div class="modal-header">
            <h2>Edit Student Information</h2>
            <button class="close-modal" @click="closeEditModal">&times;</button>
          </div>
          <form @submit="handleSubmitEdit" class="student-form">
            <div class="student-info-grid">
              <div class="info-group">
                <label>Student ID</label>
                <input type="text" v-model="formData.id" placeholder="Enter Student ID" required>
              </div>
              <div class="info-group">
                <label>First Name</label>
                <input 
                  type="text" 
                  v-model="formData.firstName" 
                  placeholder="Enter First Name" 
                  @input="validateNameInput"
                  required
                >
              </div>
              <div class="info-group">
                <label>Last Name</label>
                <input 
                  type="text" 
                  v-model="formData.lastName" 
                  placeholder="Enter Last Name" 
                  @input="validateNameInput"
                  required
                >
              </div>
              <div class="info-group">
                <label>Middle Name</label>
                <input 
                  type="text" 
                  v-model="formData.middleName" 
                  placeholder="Enter Middle Name" 
                  @input="validateNameInput"
                >
              </div>
              <div class="info-group">
                <label>Suffix</label>
                <input 
                  type="text" 
                  v-model="formData.suffix" 
                  placeholder="Enter Suffix (Optional)"
                  @input="validateNameInput"
                >
              </div>
              <div class="info-group">
                <label>Birthday</label>
                <input 
                  type="date" 
                  v-model="formData.birthday"
                  :max="'2009-12-31'"
                  required
                >
              </div>
              <div class="info-group">
                <label>Course</label>
                <select v-model="formData.course" required>
                  <option value="" disabled selected>Select Course</option>
                  <option v-for="course in courses" :key="course" :value="course">
                    {{ course }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Block</label>
                <select 
                  v-model="formData.block" 
                  required
                  :disabled="!formData.course || !formData.yearLevel"
                >
                  <option value="" disabled selected>
                    {{ !formData.course || !formData.yearLevel ? 
                      'Select Course and Year Level first' : 
                      'Select Block' 
                    }}
                  </option>
                  <option 
                    v-for="block in blockOptions" 
                    :key="block" 
                    :value="block"
                  >
                    {{ block }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Year Level</label>
                <select v-model="formData.yearLevel" required>
                  <option value="" disabled selected>Select Year Level</option>
                  <option v-for="year in yearLevels" :key="year" :value="year">
                    {{ year }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Status</label>
                <select v-model="formData.status" required>
                  <option value="" disabled selected>Select Status</option>
                  <option v-for="status in statusOptions" :key="status" :value="status">
                    {{ status }}
                  </option>
                </select>
              </div>
              <div class="info-group">
                <label>Email</label>
                <input 
                  type="email" 
                  v-model="formData.email" 
                  placeholder="Enter Student Email" 
                  required
                >
              </div>
              <div class="info-group">
                <label>Phone</label>
                <input 
                  type="tel" 
                  v-model="formData.phone" 
                  placeholder="Enter Student Phone" 
                  @input="validatePhoneInput"
                  pattern="[0-9+]+"
                  required
                >
              </div>
            </div>
            <div class="form-actions">
              <button type="button" class="cancel-btn" @click="closeEditModal">Cancel</button>
              <button type="submit" class="submit-btn">Save Changes</button>
            </div>
          </form>
        </div>
      </div>



      <!-- DELETE STUDENT MODAL -->
      <ConfirmationModal 
        :is-open="isConfirmationModalOpen" 
        :title="confirmationData.title"
        :item-name="confirmationData.itemName" 
        :item-info="confirmationData.itemInfo"
        placeholder-text="Enter Student ID to Confirm Deletion"
        :show-status-selection="true"
        :status-options="deleteStatusOptions"
        @close="isConfirmationModalOpen = false"
        @confirm="handleConfirmDelete" 
      />

      <!-- UNSAVED CHANGES MODAL -->
      <UnsavedChangesModal 
        :is-open="isUnsavedChangesModalOpen"
        @close="handleUnsavedChanges(false)"
        @confirm="handleUnsavedChanges(true)"
      />
    </section>
  </main>
</template>
