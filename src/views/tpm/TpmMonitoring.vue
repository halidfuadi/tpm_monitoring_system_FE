<template>
  <div class="container-fluid">
    <Toaster position="top-center" closeButton />
    <ModalPic :isShow="isShow" :incharge_id="incharge_id" :machine_nm="machine_nm" :plan_check_dt="plan_check_dt"
      :schedule_id="schedule_id" @showChanges="showChanges(state)" />
    <SearchBar @getSchedules="getSchedules" />
    <DropBarDelay/>
    <StatusTpm :filter="filter" />
    <CCard>
      <CCardBody>
        <CRow>
          <CCol class="tableFixHead">
            <table class="table table-bordered table-striped">
              <thead>
                <tr>
                  <th rowspan="2">No</th>
                  <th class="w100-line text-center" rowspan="2" style="font-size: 15px;">Line</th>
                  <th class="w200-mc" rowspan="2" style="font-size: 15px;">Machine</th>
                  <th class="w300-item-check text-center" rowspan="2" style="font-size: 15px;">Item Check</th>
                  <th class="text-center" rowspan="2" colspan="2" style="font-size: 15px;">Periodic</th>
                  <th class="text-center" rowspan="2" style="font-size: 15px;">Incharge</th>
                  <th class="text-center" rowspan="2" style="font-size: 15px;">PIC</th>
                  <th class="text-center" :colspan="31" style="font-size: 15px;">{{ 'Schedule' }}</th>
                  <th class="text-center w100-line" rowspan="2" style="font-size: 15px;">Next Check</th>
                </tr>
                <tr>
                  <td class="w40-date text-center" v-for="i in 31" :key="i">
                    {{ i }}
                  </td>
                </tr>
              </thead>
              <tbody>
                <template v-if="Object.keys(schedules).length > 0">
                  <tr v-for="(schedule, key, ipar) in schedules " :key="key">
                    <td>{{ ipar + 1 }}</td>
                    <td>{{ schedule[0].line_nm }}</td>
                    <td>{{ schedule[0].machine_nm }}</td>
                    <td>{{ schedule[0].itemcheck_nm }}</td>
                    <td>{{ schedule[0].val_periodic }}</td>
                    <td class="text-center">
                      {{ schedule[0].period_nm }}
                    </td>
                    <td class="text-center">{{ schedule[0].incharge_nm }}</td>
                    <td v-if="schedule[0].checkers.length > 0">
                      <template v-for="user in schedule[0].checkers" :key="user.user_id">
                        <CButton color="success" size="sm" style="z-index: 1;">
                          {{ user.user_nm }}
                        </CButton>
                      </template>
                    </td>
                    <td v-else>
                      <CButton class="btn btn-sm w-100" color="info" @click="confirmShow(schedule[0])">ASSIGN</CButton>
                    </td>
                    <template v-for="date in dates" :key="date">
                      <td v-if="schedule.find((item) => {return item.day_idx == date})">
                        <!-- BTN FOR ASSIGN PIC -->
                        <CButton size="lg" v-if="schedule.find((item) => {return item.day_idx == date}).checkers.length == 0" :style="`height: 35px ;background-color: ${schedule.find((item) => {return item.day_idx == date}).color_tag}`" @click="confirmShow(schedule.find((item) => {return item.day_idx == date}),)"></CButton>
                        <!-- BTN FOR EXECUTION -->
                        <CButton size="lg" v-else :style="`height: 35px; background-color: ${schedule.find((item) => {return item.day_idx == date}).color_tag}`" @click="executionPage(schedule.find((item) => {return item.day_idx == date}),)"></CButton>
                      </td>
                      <td v-else></td>
                    </template>
                    <td>{{ schedule[0].next_check.split('T')[0] }}</td>
                  </tr>
                </template>
                <template v-else>
                  <tr>
                    <td class="text-center" colspan="37">
                      <b>Tidak Ada Data</b>
                    </td>
                  </tr>
                </template>
              </tbody>
            </table>
          </CCol>
        </CRow>
      </CCardBody>

      <CCardFooter>
        <CRow class="justify-content-between">
          <CCol lg="2">
            <div class="input-group mb-3">
              <div class="input-group-prepend">
                <span class="input-group-text">Limit</span>
              </div>
              <select class="form-control" v-model="filtered.rowsPerPage">
                <option v-for="limit in limitOpts" :key="limit.label" :value="limit.value">
                  {{ limit.label }}
                </option>
              </select>
            </div>
          </CCol>
          <CCol lg="3">
            <ul class="pagination m-0">
              <li class="page-item disabled">
                <button class="page-link">Previous</button>
              </li>
              <li v-for="page in pages" class="page-item">
                <button class="page-link">1</button>
              </li>
              <li class="page-item">
                <button class="page-link">Next</button>
              </li>
            </ul>
          </CCol>
        </CRow>
      </CCardFooter>
    </CCard>
  </div>
</template>

<script>
import api from '@/apis/CommonAPI'
import { Toaster } from 'vue-sonner'
import ModalPic from '@/components/Tpm/ModalPic'
import SearchBar from '@/components/Tpm/SearchBar'
import StatusTpm from '../../views/charts/StatusTpm.vue'
import DropBarDelay from '../../components/Tpm/DropBarDelay.vue'
import { CButton } from '@coreui/vue'

export default {
  name: 'TpmMonitoring',
  data() {
    return {
      filtered: {
        currentPage: 1,
      },
      rowsPerPage: 5,
      maxVisible: 5,
      modelValue: 10,

      isShow: false,
      filter: null,
      schedule_id: null,
      schedules: [],
      incharge_id: null,
      machine_nm: null,
      plan_check_dt: null,
      year: null,
      month: null,
      limitOpts: [
        {
          label: 5,
          value: 5,
        },
        {
          label: 10,
          value: 10,
        },
        {
          label: 100,
          value: 100,
        },
        {
          label: 'All',
          value: -1,
        },
      ],
      months: ['January','February','March','April','May','June','July','August','September','October','November','December',],
    }
  },
  computed: {
    pages() {
      function range(start, end) {
        return Array.from(Array(end - start + 1), (_, i) => i + start)
      }

      const max = this.maxVisible
      const middle = Math.floor(this.maxVisible / 2)
      const pageNum = Math.ceil(this.rowsNumber / this.rowsPerPage)

      if (pageNum < max) {
        return range(1, pageNum)
      }

      let start = this.modelValue - middle
      let end = this.modelValue + middle

      // If we're close to the end
      if (this.modelValue >= pageNum - middle) {
        start = pageNum - max + 1
        end = pageNum
      }

      return range(Math.max(1, start), Math.max(end, max))
    },
  },
  methods: {
    onPageClick(page) {
      this.$emit('update:modelValue', page)
    },
    onPageBack() {
      if (this.modelValue - 1 >= 1) {
        this.onPageClick(this.modelValue - 1)
      }
    },
    onPageForward() {
      if (
        this.modelValue + 1 <=
        Math.ceil(this.rowsNumber / this.rowsPerPage)
      ) {
        this.onPageClick(this.modelValue + 1)
      }
    },
    async getSchedules(filter) {
      try {
        let month = filter.split('=')[1].split('-')[1]
        let year = filter.split('=')[1].split('-')[0]
        this.generateDate(month, year)
        this.filter = filter
        let schedules = await api.get(`/tpm/schedules`, '?' + filter)

        console.log(schedules)
        this.schedules = schedules.data.data
      } catch (error) {
        console.log(error)
      }
    },
    generateDate(month, year) {
      let monthIdx = this.months.indexOf(month)
      let totalDate = new Date(year, monthIdx + 1, 0, 23, 59, 59).getDate()
      this.dates = []
      console.log(new Date(year, monthIdx + 1, 0, 23, 59, 59))
      for (let index = 1; index <= totalDate; index++) {
        this.dates.push(index)
      }
    },
    showChanges(state) {
      this.isShow = state
      this.getSchedules(this.filter)
    },

    async confirmShow(schedule) {
      await this.showChanges(true)
      console.log(schedule)
      this.machine_nm = schedule.machine_nm
      this.incharge_id = schedule.incharge_id
      this.schedule_id = schedule.schedule_id
      this.plan_check_dt = schedule.plan_check_dt
      this.getSchedules(this.filter)
    },

    executionPage(schedule) {
      this.$router.push(`monitoring/${schedule.schedule_id}`)

    },
  },
  components: {
    SearchBar,
    StatusTpm,
    ModalPic,
    Toaster,
    DropBarDelay,
  },
  watch:{
    getSchedules(){
      this.getSchedules(this.filter)}
  }
}
</script>

<style scoped>
.w100-line {
  min-width: 100px;
}

.w40-date {
  min-width: 40px;
  z-index: -1 !important;
}

.w300-item-check {
  min-width: 300px;
}

.w200-mc {
  min-width: 200px;
}

th,
td {
  border: 1px solid black;
  border-collapse: collapse;
  background: white;
}

thead {
  top: 0;
  position: sticky;
  z-index: 1;
}

.tableFixHead {
  overflow-y: auto;
  height: 800px;
  z-index: 1;
}

tr td:nth-child(1),
tr td:nth-child(2),
tr td:nth-child(3),
tr td:nth-child(4),
tr th:nth-child(1),
tr th:nth-child(2),
tr th:nth-child(3),
tr th:nth-child(4) {
  position: sticky;
  left: 0;
}

tr th:nth-child(2),
tr td:nth-child(2) {
  left: 30px;
}

tr th:nth-child(3),
tr td:nth-child(3) {
  left: 100px;
}

tr th:nth-child(4),
tr td:nth-child(4) {
  left: 200px;
}
</style>
