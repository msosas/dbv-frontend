<template>
    <q-page class="flex justify-center" >
        <div style="width: 80%;">
            <h3 style="text-align: center;">
              Archivos Cambiados
            </h3>
            <div style="text-align: center;">
              <span><q-btn icon="autorenew" color="dark" label="Actualizar" @click="refresh()"/></span>
              <span><q-btn icon="done" color="positive" label="Commit" @click="goToCommit()"/></span>
              <span><q-btn icon="clear_all" color="deep-purple" label="Descartar" @click="checkout()"/></span>
            </div>
            <q-list>
              <q-list-header>
                <span>Total: {{ files.length }}</span>
              </q-list-header>
              <q-item
                :class="{ 'bg-warning': 'M' === file.type, 'bg-negative': 'D' === file.type, 'bg-positive': '?' === file.type  }"
                v-for="file in files"
                :key="file.change.key"
              >
                <q-item-main :label="file.change" />
                <q-item-side right>
                  <q-btn round size="sm" icon="visibility" @click="showDifferences(file.change)"/>
                  <q-btn round size="sm" icon="delete" @click="deleteFile(file.change)" />
                </q-item-side>
              </q-item>
            </q-list>
            <q-modal v-model="diffOpened">
              <h4 style="text-align: center;">Diferencias</h4>
              <p
                style="padding-left: 2%;"
                v-for="change in changes"
                :key="change"
                :class="{ 'new-line': change[0] === '+' || newFile, 'removed-line': change[0] === '-' }"
                v-html="change"
              >
              {{ change }}
              </p>
              <div style="text-align: center;">
                <q-btn
                  color="blue-grey-8"
                  @click="diffOpened = false"
                  label="Close"
                />
              </div>
            </q-modal>
        </div>
    </q-page>
</template>
<style type="text/css">
  .new-line {
    color: green;
  }
  .removed-line {
    color: red;
  }
</style>
<script>
var baseUrl = 'http://localhost:4001'
import axios from 'axios'
import { Notify, Loading, QSpinnerPuff } from 'quasar'
export default {
  data () {
    return {
      files: [],
      changes: '',
      diffOpened: false,
      newFile: false
    }
  },
  methods: {
    updateSchema () {
      Loading.show({ delay: 0 })
      axios.get(baseUrl + '/update_schema')
        .then(({ data }) => {
          Notify.create({
            message: 'Schema Local actualizado',
            color: 'positive'
          })
          Loading.hide()
        })
        .catch(error => {
          Notify.create('Ocurrió un error, no se puede actualizar el schema')
          console.log(error)
          Loading.hide()
        })
    },
    goToCommit () {
      this.$router.push('/commit')
    },
    gitStatus () {
      axios.get(baseUrl + '/status_raw')
        .then(({data}) => {
          this.files = data
          Loading.hide()
        })
        .catch(error => {
          Notify.create('Ocurrió un error, no se puede hacer git status')
          console.log(error)
          Loading.hide()
        })
    },
    showDifferences (file) {
      file = file.substring(1)
      Loading.show({ delay: 0 })
      this.diffOpened = true
      axios.get(baseUrl + '/differences?file=' + file)
        .then(({data}) => {
          if (data[0][0] + data[0][1] + data[0][2] + data[0][3] !== 'diff') {
            this.newFile = true
          } else {
            this.newFile = false
          }
          this.changes = data
          Loading.hide()
        })
        .catch(error => {
          Notify.create('Ocurrió un error, no se pueden obtener las diferencias')
          console.log(error)
          Loading.hide()
        })
    },
    generateFiles (callback) {
      axios.get(baseUrl + '/generate_files')
        .then(({ data }) => {
          console.log(data)
          callback(null, 'OK')
        })
        .catch(error => {
          Notify.create('Ocurrió un error, no se pueden generar los archivos')
          console.log(error)
          Loading.hide()
        })
    },
    refresh () {
      var that = this
      this.files = []
      Loading.show({
        spinner: QSpinnerPuff,
        spinnerSize: 250,
        delay: 0,
        message: 'Generando archivos...'
      })
      this.$nextTick(function () {
        that.generateFiles(function () {
          setTimeout(function () {
            that.gitStatus()
            Loading.hide()
          }, 100)
        })
      })
    },
    checkout () {
      axios.get(baseUrl + '/checkout')
        .then(({ data }) => {
          Notify.create({
            message: 'Cambios Descartados',
            color: 'positive'
          })
          this.$nextTick(function () {
            this.gitStatus()
          })
          this.updateSchema()
        })
        .catch(error => {
          Notify.create('Ocurrió un error, no se pueden descartar los cambios')
          console.log(error)
        })
    },
    deleteFile (file) {
      axios.delete(baseUrl + '/delete_file', { data: { 'file': file.substring(1) } })
        .then(({ data }) => {
          Notify.create({
            message: 'Archivo borrado',
            color: 'positive'
          })
          this.$nextTick(function () {
            this.gitStatus()
          })
        })
        .catch(error => {
          Notify.create('Ocurrió un error, no se puede borrar el archivo')
          console.log(error)
        })
    }
  },
  created () {
    this.refresh()
  }
}
</script>
