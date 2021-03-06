<template>
  <v-col auto>
    <v-data-table
      :key="list.length"
      :headers="headers"
      :items="list"
      :mobile-breakpoint="NaN"
      :no-data-text="$t('label.data_empty')"
      :items-per-page="listQuery.limit"
      hide-default-footer
    >
      <template v-slot:item="{ item, index }">
        <tr>
          <td>{{ getTableRowNumbering(index) }}</td>
          <td>{{ item.name }}</td>
          <td>{{ item.age }} Th</td>
          <td>
            <div v-if="item.gender === 'P'">
              {{ $t('label.female_initials') }}
            </div>
            <div v-else>
              {{ $t('label.male_initials') }}
            </div>
          </td>
          <td><status :status="item.status" /> </td>
          <td>
            <div v-if=" item.last_history.stage === '0'">
              {{ $t('label.process') }}
            </div>
            <div v-else>
              {{ $t('label.done') }}
            </div>
          </td>
          <td>
            <div v-if=" item.last_history.final_result === '0'">
              {{ $t('label.negatif') }}
            </div>
            <div v-else-if=" item.last_history.final_result === '1'">
              {{ $t('label.recovery') }}
            </div>
            <div v-else-if=" item.last_history.final_result === '2'">
              {{ $t('label.dead') }}
            </div>
            <div v-else>
              -
            </div>
          </td>
          <td>{{ item.author.username }}</td>
          <td v-if="roles[0] === 'faskes'">
            {{ formatDatetime(item.createdAt, 'DD MMMM YYYY') }}
          </td>
          <td v-if="roles[0] === 'faskes'" class="adjust-width">
            <span
              :class="{'pending': item.verified_status === 'pending', 'declined': item.verified_status === 'declined'}"
              class="pa-2 font-weight-bold"
            >
              {{ item.verified_status === 'pending' ? $t('label.waiting_for_verification') : $t('label.case_rejected') }}
            </span>
          </td>
          <td v-else-if="item.verified_status !== 'declined'">{{ item.last_history.createdAt ? timeRemain(item.updatedAt) : '-' }}</td>
          <td v-else-if="tableHeaders.length > 8">&nbsp;</td>
          <td v-if="tableHeaders.length > 8">
            <v-card-actions>
              <v-menu
                :close-on-content-click="false"
                :nudge-width="100"
                :nudge-left="220"
                :nudge-top="40"
                offset-y
              >
                <template v-slot:activator="{ on }">
                  <v-btn
                    v-if="showActionButton(item.verified_status)"
                    class="ma-1 action-btn"
                    color="#828282"
                    tile
                    outlined
                    v-on="on"
                  >
                    {{ $t('label.choose_action') }}
                    <v-icon class="action-icon" right>
                      mdi-menu-down
                    </v-icon>
                  </v-btn>
                  <div v-else>
                    &nbsp;
                  </div>
                </template>
                <v-card>
                  <div v-if="roles[0] === 'faskes'">
                    <v-list-item @click="handleDetail(item._id)">
                      {{ item.verified_status === 'declined' ? $t('label.fix_case') : $t('label.view_case_detail') }}
                    </v-list-item>
                    <v-list-item v-if="item.verified_status !== 'declined'" @click="handleEditCase(item._id)">
                      {{ $t('label.profile_update') }}
                    </v-list-item>
                    <v-list-item v-if="item.verified_status !== 'declined'" @click="handleEditHistoryCase(item._id)">
                      {{ $t('label.update_history') }}
                    </v-list-item>
                    <v-list-item @click="handleDeleteCase(item)">
                      <span class="delete">{{ $t('label.delete_case') }}</span>
                    </v-list-item>
                  </div>
                  <div v-else>
                    <v-list-item @click="handleDetail(item._id)">
                      {{ $t('label.verification_process') }}
                    </v-list-item>
                  </div>
                </v-card>
              </v-menu>
            </v-card-actions>
          </td>
        </tr>
      </template>
    </v-data-table>
    <dialog-delete
      :dialog="dialog"
      :data-deleted="dataDelete"
      :dialog-delete.sync="dialog"
      :delete-date.sync="dataDelete"
      :store-path-delete="`reports/deleteReportCase`"
      :store-path-get-list="`reports/listReportCase`"
      :list-query="listQuery"
    />
  </v-col>
</template>
<script>
import { mapGetters } from 'vuex'
import moment from 'moment'
import { formatDatetime } from '@/utils/parseDatetime'
export default {
  name: 'VerificationTable',
  props: {
    listKasus: {
      type: Array,
      default: null
    },
    query: {
      type: Object,
      default: null
    },
    tableHeaders: {
      type: Array,
      default: null
    }
  },
  data() {
    return {
      list: this.listKasus,
      headers: this.tableHeaders,
      listQuery: this.query,
      dialog: false,
      dataDelete: null,
      verificationQuery: {
        'id': '',
        'data': {
          'verified_status': '',
          'verified_comment': ''
        }
      }
    }
  },
  computed: {
    ...mapGetters('user', [
      'roles'
    ])
  },
  watch: {
    listKasus(value) {
      this.list = value
    },
    tableHeaders(value) {
      this.headers = value
    },
    query(value) {
      this.listQuery = value
    }
  },
  methods: {
    formatDatetime,
    getTableRowNumbering(index) {
      return ((this.listQuery.page - 1) * this.listQuery.limit) + (index + 1)
    },
    timeRemain(value) {
      const now = moment()
      const maxVerifiedDate = moment(value).add(1, 'days')

      // output otomatis mengeluarkan kalimat waktu. Contoh: dalam 3 jam
      return (maxVerifiedDate > now) ? now.to(maxVerifiedDate) : '-'
    },
    async handleDetail(id) {
      this.$emit('update:verificationQuery', this.verificationQuery)
      const response = await this.$store.dispatch('reports/detailReportCase', id)
      if (response.data.verified_status === 'verified') {
        this.$emit('update:showFailedDialog', true)
      } else {
        this.$emit('update:caseDetail', response.data)
        this.$emit('update:showVerificationForm', true)
      }
    },
    async handleEditCase(id) {
      await this.$router.push(`/laporan/edit-case/${id}`)
    },
    async handleEditHistoryCase(id) {
      await this.$router.push(`/laporan/edit-history-case/${id}`)
    },
    async handleDeleteCase(item) {
      this.dialog = true
      this.dataDelete = await item
    },
    showActionButton(status) {
      if (this.roles[0] === 'dinkeskota') {
        return status === 'pending'
      } else {
        return true
      }
    }
  }
}
</script>
<style>
.delete {
  color: #fb2c2c !important;
}
.action-btn {
  text-transform: none !important;
  height: 30px !important;
  min-width: 80px !important;
}
.action-icon {
  color: #009D57 !important;
  font-size: 2rem !important;
}
.adjust-width {
  width: 20%;
}
</style>
