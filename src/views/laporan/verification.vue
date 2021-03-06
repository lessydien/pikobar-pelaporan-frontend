<template>
  <div>
    <v-card class="disclaimer pa-7 mb-5" width="100%">
      <v-row class="mx-0 mb-2 headline">
        <div class="font-white">
          {{ $t('label.total_new_cases') }} : {{ totalItem }}
        </div>
      </v-row>
      <v-row class="mx-0 mb-0">
        <span class="font-white">
          {{ roles[0] === 'faskes' ? $t('label.medical_facility_verification_info_1') : $t('label.verification_info') }}
        </span>
        <br>
      </v-row>
      <v-row v-if="roles[0] === 'faskes'" class="mx-0">
        <span class="font-white">
          {{ $t('label.medical_facility_verification_info_2') }}
        </span>
      </v-row>
    </v-card>
    <v-card outlined>
      <v-container>
        <v-row class="filter-row mt-5" justify="center">
          <v-col cols="12">
            <v-text-field
              v-model="listQuery.search"
              solo
              :label="$t('label.search')"
              prepend-inner-icon="search"
            />
          </v-col>
        </v-row>
        <v-row justify="center">
          <v-col cols="12" class="reduce-padding-top">
            <address-region
              :disabled-district="disabledDistrict"
              :district-code="listQuery.address_district_code"
              :district-name="district_name_user"
              :code-district.sync="listQuery.address_district_code"
              :sub-district-code="listQuery.address_subdistrict_code"
              :code-sub-district.sync="listQuery.address_subdistrict_code"
              :village-code="listQuery.address_village_code"
              :code-village.sync="listQuery.address_village_code"
              :village-name="villageName"
              :name-village.sync="villageName"
              :disabled-address="false"
              :required-address="false"
              :is-label="true"
            />
          </v-col>
        </v-row>
        <v-row class="filter-row" justify="center">
          <v-col cols="12" sm="4">
            <v-label class="title">{{ $t('label.reporting_sources') }}:</v-label>
            <v-text-field
              v-if="roles[0] === 'faskes'"
              v-model="listQuery.author"
              solo-inverted
              disabled
            />
            <v-autocomplete
              v-else
              v-model="listQuery.author"
              :items="listMedicalFacility"
              solo
              item-text="fullname"
              item-value="_id"
            />
          </v-col>
          <v-col cols="12" sm="4">
            <v-label class="title">{{ $t('label.criteria') }}:</v-label>
            <v-select
              v-model="listQuery.status"
              :items="stagesList"
              solo
              item-text="label"
              item-value="value"
            />
          </v-col>
          <v-col cols="12" sm="4">
            <br>
            <v-row>
              <v-col class="reduce-padding-top">
                <v-btn
                  block
                  color="#4f4f4f"
                  class="btn-reset"
                  @click="onReset"
                >
                  {{ $t('label.reset') }}
                </v-btn>
              </v-col>
              <v-col class="reduce-padding-top">
                <v-btn
                  block
                  color="success"
                  class="btn-cari"
                  @click="handleSearch"
                >
                  {{ $t('label.look_for_it') }}
                </v-btn>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-container>
      <v-row class="mx-0 mt-5">
        <v-tabs
          v-model="tab"
          class="elevation-2"
          background-color="white"
          active-class="active-class"
          color="red"
          hide-slider
        >
          <v-tab @click="onTabChanges('pending,declined')">{{ tabLabel[0] }}</v-tab>
          <v-tab @click="onTabChanges('pending')">{{ tabLabel[1] }}</v-tab>
          <v-tab @click="onTabChanges('declined')">{{ tabLabel[2] }}</v-tab>
          <v-tab-item v-for="(tabItem, index) in tabLabel" :key="index">
            <v-row>
              <verification-table
                :table-headers="headers"
                :list-kasus="listKasus"
                :query="listQuery"
                :show-failed-dialog.sync="showFailedDialog"
                :show-verification-form.sync="showVerificationForm"
                :case-detail.sync="caseDetail"
                :verification-query="verificationQuery"
              />
            </v-row>
          </v-tab-item>
        </v-tabs>
      </v-row>
    </v-card>
    <pagination
      :total="totalList"
      :page.sync="listQuery.page"
      :limit.sync="listQuery.limit"
      :on-next="onNext"
    />
    <verification-form
      :show-form="showVerificationForm"
      :show.sync="showVerificationForm"
      :show-confirmation.sync="showConfirmation"
      :case-data="caseDetail"
      :query-data="verificationQuery"
      :query.sync="verificationQuery"
      :refresh-page.sync="isRefresh"
    />
    <verification-confirmation
      :show-dialog="showConfirmation"
      :show.sync="showConfirmation"
      :query-data="verificationQuery"
      :query.sync="verificationQuery"
      :submit-data.sync="isSubmit"
    />
    <dialog-failed
      :show-dialog="showFailedDialog"
      :show.sync="showFailedDialog"
      :title="$t('label.verification_expired_title')"
      :message="$t('label.verification_expired_message')"
    />
  </div>
</template>

<script>
import { mapGetters } from 'vuex'

export default {
  name: 'LaporanList',
  data() {
    return {
      headers: [
        { text: '#', value: '_id', sortable: false },
        { text: this.$t('label.name').toUpperCase(), value: 'name' },
        { text: this.$t('label.age').toUpperCase(), value: 'age' },
        { text: this.$t('label.gender_abbreviation').toUpperCase(), value: 'gender' },
        { text: this.$t('label.criteria').toUpperCase(), value: 'criteria' },
        { text: this.$t('label.stages').toUpperCase(), value: 'stage' },
        { text: this.$t('label.results').toUpperCase(), value: 'final_result' },
        { text: this.$t('label.reporter').toUpperCase(), value: 'author' }
      ],
      listQuery: {
        address_district_code: '',
        address_subdistrict_code: '',
        address_village_code: '',
        status: '',
        author: '',
        page: 1,
        limit: 30,
        search: '',
        verified_status: ''
      },
      medicalFacilityListQuery: {
        role: 'faskes',
        code_district_city: '',
        search: ''
      },
      stagesList: [
        'OTG',
        'ODP',
        'PDP',
        'POSITIF'
      ],
      failedDialog: false,
      disabledDistrict: true,
      listMedicalFacility: [],
      villageName: '',
      dialogForm: true,
      caseDetail: null,
      showVerificationForm: false,
      showConfirmation: false,
      showFailedDialog: false,
      isSubmit: false,
      isRefresh: false,
      tab: null,
      tabLabel: [this.$t('label.all'), this.$t('label.waiting_for_verification'), this.$t('label.case_rejected')],
      totalItem: null,
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
    ...mapGetters('reports', [
      'listKasus',
      'totalList'
    ]),
    ...mapGetters('user', [
      'roles',
      'district_user',
      'district_name_user'
    ])
  },
  watch: {
    'listQuery.search': {
      handler: function(value) {
        if ((value !== undefined) && (value.length !== 0 || value.length >= 3)) {
          this.listQuery.page = 1
          this.handleSearch()
        }
      },
      immediate: true
    },
    async isSubmit(value) {
      if (value) {
        const response = await this.$store.dispatch('reports/verifyCase', this.verificationQuery)
        if (response.status === 200 || response.status === 201) {
          this.handleSearch()
          await this.$store.dispatch('toast/successToast', this.verificationQuery.data.verified_status === 'verified' ? this.$t('success.verification_success') : this.$t('success.rejection_success'))
        }
        this.isSubmit = false
      }
    },
    isRefresh(value) {
      if (value) {
        this.handleSearch()
        this.isRefresh = false
      }
    }
  },
  async mounted() {
    if (this.roles[0] === 'faskes') {
      this.headers.push(
        { text: this.$t('label.input_date').toUpperCase(), value: 'inputDate' },
        { text: this.$t('label.status').toUpperCase(), value: 'status' },
        { text: this.$t('label.action').toUpperCase(), value: 'action', sortable: false }
      )
      this.listQuery.verified_status = 'pending,declined'
    } else {
      this.headers.push(
        { text: this.$t('label.auto_verification_deadline').toUpperCase(), value: 'createdAt' },
        { text: this.$t('label.action').toUpperCase(), value: 'action', sortable: false }
      )
      this.listQuery.verified_status = 'pending,declined'
    }
    const response = await this.$store.dispatch('reports/countVerificationCase')
    this.totalItem = response.data.PENDING + response.data.DECLINED
    if (this.roles[0] !== 'faskes') {
      this.listQuery.sort = { 'updatedAt': 'asc' }
    }
    await this.$store.dispatch('reports/listReportCase', this.listQuery)
    this.listQuery.address_district_code = this.district_user
    this.medicalFacilityListQuery.code_district_city = this.district_user
    const responseMedList = await this.$store.dispatch('reports/listMedicalFacility', this.medicalFacilityListQuery)
    this.listMedicalFacility = responseMedList.data
  },
  methods: {
    async handleSearch() {
      await this.$store.dispatch('reports/countVerificationCase')
      await this.$store.dispatch('reports/listReportCase', this.listQuery)
    },
    async onNext() {
      await this.$store.dispatch('reports/listReportCase', this.listQuery)
    },
    onReset() {
      this.listQuery.search = ''
      this.listQuery.final_result = ''
      this.listQuery.status = ''
      if (this.roles[0] !== 'faskes') {
        this.listQuery.author = ''
      }
      this.listQuery.address_subdistrict_code = ''
      this.listQuery.address_village_code = ''
      this.villageName = ''
      if (this.roles[0] === 'superadmin') {
        this.listQuery.address_district_code = ''
      }
      this.$store.dispatch('reports/listReportCase', this.listQuery)
    },
    onTabChanges(value) {
      const ids = this.headers.length
      if (this.roles[0] === 'dinkeskota' && value === 'declined') {
        this.headers.splice(8, 2)
      } else if (ids === 8) {
        this.headers.push(
          { text: this.$t('label.auto_verification_deadline').toUpperCase(), value: 'createdAt' },
          { text: this.$t('label.action').toUpperCase(), value: 'action', sortable: false }
        )
      }
      this.listQuery.verified_status = value
      this.handleSearch()
    }
  }
}
</script>
<style>
  .title {
    font-size: 2rem;
    text-transform: uppercase;
    margin-left: 30px;
    color: #828282;
  }
  .align-right {
    text-align: right;
    padding-right: 50px;
  }
  .margin-right {
    margin-right: 8px;
  }
  .margin-left {
    margin-left: 8px;
  }
  .info-message {
    background-color: #d2eaff;
    border-radius: 10px;
    color: #2f7fed;
  }
  .reduce-padding-top {
    padding-top: 0px !important;
  }
  .reduce-padding-right {
    padding-right: 6px;
  }
  .reduce-padding-left {
    padding-left: 6px;
  }
  .btn-reset {
    height: 46px !important;
    color: white !important;
  }
  .btn-cari {
    height: 46px !important;
  }
  .verif-btn {
    background-color: white;
    border-radius: 10px;
    border-color: grey;
    border-style: solid;
    border-width: thin;
    color: grey;
    cursor: pointer;
  }
  .verif-btn:hover {
    background-color: grey;
    border-radius: 10px;
    border-color: grey;
    border-style: solid;
    border-width: thin;
    color: white;
    cursor: pointer;
  }
  .pending {
    color: white;
    border-radius: 10px;
    background-color: #2a58ff;
  }
  .declined {
    color: white;
    border-radius: 10px;
    background-color: #f91717;
  }
  .disclaimer {
    background: linear-gradient(78.54deg, #27AE60 0%, #6FCF97 100%) !important;
  }
  .font-white {
    color: white;
  }
</style>
