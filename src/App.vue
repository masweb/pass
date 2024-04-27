<script setup lang="ts">
import { writeText  } from '@tauri-apps/api/clipboard'
import { readTextFile, writeTextFile, BaseDirectory } from '@tauri-apps/api/fs'
import { ref, onMounted, onBeforeUnmount, computed, nextTick } from 'vue'

import { SkeletonLoader } from "vue3-loading-skeleton"
import {CopyIcon, EyeIcon, EyeClosedIcon, CirclePlusIcon,
  SettingsIcon, TrashIcon, DeviceFloppyIcon} from 'vue-tabler-icons'

interface iPass {
  id: number
  name: string
  emailus: string
  pass: string
  isEdit: boolean
}

const showSecrets = ref(false)
const filterName = ref('')
const filterEmail = ref('')
const filterPassword = ref('')
const showCopied = ref(false)
const messsage = ref('')
const showDeleteModal = ref(false)
const passToDelete = ref({} as iPass)
const showNewPass = ref(false)
const focusNewPass = ref(false)

const newService = ref('')
const newUserEmail = ref('')
const newPassword = ref('')

const passes = ref([] as iPass[])

onMounted(() => {
  readPasses()

  const escFunction = (event: KeyboardEvent) => {
    if (event.key === 'Escape') closeAllEdits()
  }
  window.addEventListener('keydown', escFunction)
  onBeforeUnmount(() => {
    window.removeEventListener('keydown', escFunction)
  })
})

const closeAllEdits = async  () => {
  passes.value.forEach(pass => pass.isEdit = false)
  showSecrets.value = false
  showNewPass.value = false
  showDeleteModal.value = false
  newService.value = ''
  newUserEmail.value = ''
  newPassword.value = ''
  filterName.value = ''
  filterEmail.value = ''
  filterPassword.value = ''
  await nextTick()

}


const readPasses = async () => {
  const readPasses = await readTextFile('passes.json', { dir: BaseDirectory.Resource })
  passes.value = JSON.parse(readPasses) as iPass[]
}

const savePasses = async () => {
  await closeAllEdits()
  await writeTextFile({ path: 'passes.json', contents: JSON.stringify(passes.value) }, { dir: BaseDirectory.Resource })
  await showMesssage('Guardado!')
}

const deletePass = async (id: number) => {
  const passToDelete = passes.value.findIndex(pass => pass.id === id)
  passes.value.splice(passToDelete, 1)
  showDeleteModal.value = false
  await writeTextFile({ path: 'passes.json', contents: JSON.stringify(passes.value) }, { dir: BaseDirectory.Resource })
  await showMesssage('Eliminado!')
  await closeAllEdits()
}

const newPass = async () => {
  if(!newService.value || !newUserEmail.value || !newPassword.value) return await showMesssage('Faltan datos')
  passes.value.push({
    id: passes.value.length + 1,
    name: newService.value,
    emailus: newUserEmail.value,
    pass: newPassword.value,
    isEdit: false
  })
  showNewPass.value = false
  await writeTextFile({ path: 'passes.json', contents: JSON.stringify(passes.value) }, { dir: BaseDirectory.Resource })
  await showMesssage('Guardado!')
  await closeAllEdits()
}

const fshowDeleteModal = async (id: number) => {
  passToDelete.value = passes.value.find(pass => pass.id === id)
  showDeleteModal.value = true
}

const filteredPasses = computed(() => {
  return passes.value.filter(pass => {
    return pass.name.includes(filterName.value) &&
        pass.emailus.includes(filterEmail.value) &&
        pass.pass.includes(filterPassword.value)
  })
})


const showMesssage = async (message:string) => {
  messsage.value = message
  showCopied.value = true
  setTimeout(() => showCopied.value = false, 2000)
}


const copyToClipboard = async (text: string) => {
  await writeText(text)
  await showMesssage('Copiado!')
}

const generatePassword = () => {
  const longitud = 20
  const digitos = '0123456789'
  const letrasMayusculas = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'
  const letrasMinusculas = 'abcdefghijklmnopqrstuvwxyz'
  const caracteresEspeciales = '$%&#!@'
  const todosLosCaracteres = digitos + letrasMayusculas + letrasMinusculas + caracteresEspeciales

  let password = ''
  password += digitos[Math.floor(Math.random() * digitos.length)]
  password += letrasMayusculas[Math.floor(Math.random() * letrasMayusculas.length)]
  password += caracteresEspeciales[Math.floor(Math.random() * caracteresEspeciales.length)]

  for (let i = 3; i < longitud; i++)
    password += todosLosCaracteres[Math.floor(Math.random() * todosLosCaracteres.length)]

  password = password.split('').sort(() => Math.random() - 0.5).join('')

  return password
}

const openNewModal = () => {
  showNewPass.value = true
  if(focusNewPass.value) {
   nextTick(() => {
     focusNewPass.value.focus()
   })
  }
}

</script>

<template>

  <div class="modal " :class="{'is-active': showDeleteModal}" >
    <div class="modal-background"></div>
    <div class="modal-content">
      <div class="card is-small" >
        <header class="modal-card-head">
          <p class="modal-card-title"> ¿Quieres eleiminar {{passToDelete.name}}?</p>
          <button @click="showDeleteModal=false" class="delete" aria-label="close"></button>
        </header>
        <div class="card-content">
            <button @click="deletePass(passToDelete.id)" class="button is-danger">Eliminar</button>
        </div>
      </div>
    </div>
<!--    <button @click="showDeleteModal=false" class="modal-close is-large" aria-label="close"></button>-->
  </div>

  <div class="modal " :class="{'is-active': showNewPass}" >
    <div  class="modal-background"></div>
    <div class="modal-content">
      <div class="card is-small" >
        <header class="modal-card-head">
          <p class="modal-card-title">Nuevo password</p>
          <button @click="showNewPass=false" class="delete" aria-label="close"></button>
        </header>
        <section class="modal-card-body">
          <div class="field">
            <label class="label">Servicio</label>
            <div class="control">
              <input  v-model="newService"  @keyup.enter="newPass" ref="focusNewPass"
                      class="input is-family-secondary" type="text" placeholder="Servicio">
            </div>
          </div>
          <div class="field">
            <label class="label">Usuario / email</label>
            <div class="control">
              <input  v-model="newUserEmail"  @keyup.enter="newPass"  class="input" type="text" placeholder="Usuario / email">
            </div>
          </div>
          <label class="label">Contraseña</label>
          <div class="field has-addons">
            <div class="control is-expanded">
              <input v-model="newPassword" @keyup.enter="newPass"  class="input" type="text" placeholder="Password">
            </div>
            <div class="control">
              <button @click="newPassword = generatePassword()" class="button">  Generar contraseña  </button>
            </div>
          </div>

        </section>
        <footer class="modal-card-foot ">
          <div class="buttons">
            <button  @click="newPass" class="button is-success">Guardar</button>
          </div>
        </footer>
      </div>
    </div>
<!--    <button @click="showDeleteModal=false" class="modal-close is-large" aria-label="close"></button>-->
  </div>



  <div class="content">
    <div class="box">
      <div class="is-flex is-justify-content-space-between is-align-items-center">
        <div>
          <button @click="openNewModal" class="button mr-2">
            <CirclePlusIcon stroke-width="1" size="32"/>
          </button>
        </div>
        <article v-if="showCopied">
          <h2 class="subtitle">{{messsage}}</h2>
        </article>
        <div>
        </div>
        <div>
          <button @click="showSecrets = !showSecrets" class="button is-danger">
            <EyeIcon v-show="!showSecrets" stroke-width="1" size="32"/>
            <EyeClosedIcon v-show="showSecrets"e stroke-width="1" size="32"/>
          </button>
        </div>
      </div>
    </div>
    <table class="table">
      <thead>
      <tr>
        <th>Servicio</th>
        <th>Usuario / email</th>
        <th>Password</th>
        <th></th>
        <th></th>
      </tr>
      </thead>
      <thead>
      <tr>
        <th>
          <input v-model="filterName"  class="input" type="text" placeholder="Filtrar servicio"/>
        </th>
        <th>
          <input v-model="filterEmail" class="input" type="text" placeholder="Filtrar Usuario / email"/>
        </th>
        <th>
          <input v-model="filterPassword" class="input" type="text" placeholder="Filtrar password"/>
        </th>
        <th>Editar</th>
        <th>Eliminar</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="pass in filteredPasses" :key="pass.id">
        <td>
          <div class="mt-1">
            <strong>{{ pass.name }}</strong>
          </div>
        </td>
        <td>
          <div v-if="pass.isEdit">
            <input v-model="pass.emailus" @keyup.esc="pass.isEdit = false"
                   class="input is-small" type="text" placeholder="Usuario / email"/>
          </div>
          <div v-else class="is-flex is-justify-content-space-between is-align-items-center">
            <div v-if="showSecrets">{{ pass.emailus }}</div>
            <SkeletonLoader width="200" pill backgroundColor="#222" duration="10s" v-else/>
            <button @click="copyToClipboard(pass.emailus)" class="button is-small ml-2">
              <CopyIcon stroke-width="1" size="20"/>
            </button>
          </div>
        </td>
        <td>
          <div v-if="pass.isEdit">
            <input v-model="pass.pass" @keyup.esc="pass.isEdit = false"
                   class="input is-small" type="text" placeholder="Usuario / email"/>
          </div>
          <div v-else class="is-flex is-justify-content-space-between is-align-items-center">
            <div v-if="showSecrets">{{ pass.pass }}</div>
            <SkeletonLoader width="200" pill backgroundColor="#222" duration="10s" v-else/>
            <button @click="copyToClipboard(pass.pass)" class="button is-small ml-2">
              <CopyIcon stroke-width="1" size="20"/>
            </button>
          </div>
        </td>
        <td>
          <button  v-if="!pass.isEdit" @click="pass.isEdit = true" class="button is-small ml-2 is-danger">
            <SettingsIcon stroke-width="1" size="20"/>
          </button>
          <button @click="savePasses"  v-else class="button is-small ml-2 is-danger">
            <DeviceFloppyIcon stroke-width="1" size="20"/>
          </button>
        </td>
        <td>
          <button @click="fshowDeleteModal(pass.id)" class="button is-small ml-2 is-danger">
            <TrashIcon stroke-width="1" size="20"/>
          </button>
        </td>
      </tr>
      </tbody>
    </table>
  </div>
</template>

<style lang="scss">
:root {
  --bulma-primary: red;
}
</style>
