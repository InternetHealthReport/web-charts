<template>
  <div id="IHR_as-and-ixp-container" class="IHR_char-container">
    <div v-if="countryCode">
      <div>
        <h1 class="text-center">{{ headerString }} ({{ subHeader }})</h1>
        <h3 class="text-center">
          {{ interval.dayDiff() }}-day report ending on {{ reportDateFmt }}
          <date-time-picker
            :min="minDate"
            :max="maxDate"
            :value="maxDate"
            @input="setReportDate"
            hideTime
            class="IHR_subtitle_calendar"
          />
        </h3>
      </div>
      <q-list v-if="showGraphs">
        <q-expansion-item
          :label="$t('charts.countryHegemony.title')"
          caption="BGP data / APNIC population estimates"
          header-class="IHR_charts-title"
          icon="fas fa-project-diagram"
          :disable="show.hegemony_disable"
          v-model="show.hegemony"
        >
          <q-separator />
          <q-card class="IHR_charts-body">
            <q-card-section>
              <country-hegemony-chart
                :start-time="startTime"
                :end-time="endTime"
                :country-code="countryCode"
                :address-family="family"
                :fetch="fetch"
                ref="asInterdependenciesChart"
                @eyeballs="setMajorEyeballs($event)"
              />
            </q-card-section>
          </q-card>
        </q-expansion-item>

        <q-expansion-item
          :label="$t('charts.networkDelay.title')"
          caption="Traceroute data"
          header-class="IHR_charts-title"
          icon="fas fa-shipping-fast"
          v-model="show.net_delay"
          :disable="show.net_delay_disable"
        >
          <q-separator />
          <q-card class="IHR_charts-body">
            <q-card-section>
              <network-delay-chart
                group="start"
                :start-time="startTime"
                :end-time="endTime"
                :startPointNames="majorEyeballs"
                :endPointNames="['AS415169', 'CT4Amsterdam, North Holland, NL', 'CT4Singapore, Central Singapore, SG', 'CT4New York City, New York, US']"
                :eyeballThreshold="majorEyeballsThreshold"
                :fetch="majorEyeballs.length!=0"
                searchBar
                ref="networkDelayChart"
                @display="displayNetDelay"
              />
            </q-card-section>
          </q-card>
        </q-expansion-item>

        <q-expansion-item
          :label="$t('charts.disconnections.title')"
          caption="RIPE Atlas log"
          header-class="IHR_charts-title"
          icon="fas fa-plug"
          :disable="show.disco_disable"
          v-model="show.disco"
        >
          <q-separator />
          <q-card class="IHR_charts-body">
            <q-card-section>
              <disco-chart
                :streamName="asNumber"
                :start-time="startTime"
                :end-time="endTime"
                :fetch="fetch"
                :minAvgLevel="9"
                ref="ihrChartDisco"
              />
            </q-card-section>
          </q-card>
        </q-expansion-item>
        <div class="IHR_last-element">&nbsp;</div>
      </q-list>
    </div>
    <div v-else>
      <div>
        <h1 class="text-center q-pa-xl">Country Report</h1>
        <div class="row justify-center">
          <div class="col-8">
            <network-search-bar
              bg="white"
              label="grey-8"
              input="black"
              labelTxt="Enter an ASN, IXP ID, or network name (at least 3 characters)"
            />
          </div>
        </div>
      </div>
      <div class="q-pa-xl">
        <div class="row justify-center">
          <div class="col-6">
            <h3>Examples:</h3>
          </div>
        </div>
        <div class="row justify-center">
          <div class="col-3">
            <ul>
              <li>
                <router-link
                  :to="{ name: 'networks', params: { asn: 'AS2497' } }"
                  class="IHR_delikify"
                  >IIJ (AS2497)</router-link
                >
              </li>
              <li>
                <router-link
                  :to="{ name: 'networks', params: { asn: 'AS15169' } }"
                  class="IHR_delikify"
                  >Google (AS15169)</router-link
                >
              </li>
              <li>
                <router-link
                  :to="{ name: 'networks', params: { asn: 'AS2501' } }"
                  class="IHR_delikify"
                  >University of Tokyo (AS2501)</router-link
                >
              </li>
            </ul>
          </div>
          <div class="col-3">
            <ul>
              <li>
                <router-link
                  :to="{ name: 'networks', params: { asn: 'AS7922' } }"
                  class="IHR_delikify"
                  >Comcast (AS7922)</router-link
                >
              </li>
              <li>
                <router-link
                  :to="{ name: 'networks', params: { asn: 'AS25152' } }"
                  class="IHR_delikify"
                  >K-Root server (AS25152)</router-link
                >
              </li>
              <li>
                <router-link
                  :to="{ name: 'networks', params: { asn: 'IXP208' } }"
                  class="IHR_delikify"
                  >DE-CIX (IXP208)</router-link
                >
              </li>
            </ul>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import reportMixin from "@/views/mixin/reportMixin";
import CountryHegemonyChart from "@/views/charts/CountryHegemonyChart";
import DiscoChart, {
  DEFAULT_DISCO_AVG_LEVEL
} from "@/views/charts/global/DiscoChart";
import NetworkDelayChart from "@/views/charts/NetworkDelayChart";
import { AS_FAMILY, NetworkQuery } from "@/plugins/IhrApi";
import DateTimePicker from "@/components/DateTimePicker";
import NetworkSearchBar from "@/components/search_bar/NetworkSearchBar";
import { isoCountries } from "@/plugins/countryName";

const LOADING_STATUS = {
  ERROR: -3,
  EXPIRED: -2,
  NOT_FOUND: -1,
  LOADING: 0,
  LOADED: 1
};

const CHART_REFS = [
  "countryHegemonyChart",
  "networkDelayChart",
  "delayAndForwardingChart",
  "ihrChartDisco"
];

export default {
  mixins: [reportMixin],
  components: {
    CountryHegemonyChart,
    DiscoChart,
    NetworkDelayChart,
    DateTimePicker,
    NetworkSearchBar
  },
  data() {
    let addressFamily = this.$route.query.af;
    return {
      asNumber: 2497,
      addressFamily: addressFamily == undefined ? 4 : addressFamily,
      loadingStatus: LOADING_STATUS.LOADING,
      countryCode: this.$route.params.cc,
      countryName: null,
      charRefs: CHART_REFS,
      minAvgLevel: DEFAULT_DISCO_AVG_LEVEL,
      show: {
        disco: true,
        disco_disable: false,
        hegemony: true,
        hegemony_disable: false,
        net_delay: true,
        net_delay_disable: false
      },
      majorEyeballs: [],
      majorEyeballsThreshold: 10,
    };
  },
  methods: {
    pushRoute() {
      this.$router.push({
        //this.$router.replace({ query: Object.assign({}, this.$route.query, { hege_dt: clickData.points[0].x, hege_tb: table }) });
        query: Object.assign({}, this.$route.query, {
          af: this.family,
          last: this.interval.dayDiff(),
          date: this.$options.filters.ihrUtcString(this.interval.end, false),
        })
      });
      this.loadingStatus = LOADING_STATUS.LOADED;
      this.fetch = true;
    },
    displayNetDelay(displayValue) {
      this.show.net_delay = displayValue;
      this.$nextTick(function() {
        this.show.net_delay_disable = !displayValue;
      });
    },
    setMajorEyeballs(asns){ 
        var tmp=[]
        asns.forEach(elem => { 
            tmp.push('AS4'+elem)
        })
        this.majorEyeballs = tmp
    }
  },
  mounted() {
  },
  computed: {
    family() {
      return this.addressFamily == 6 ? AS_FAMILY.v6 : AS_FAMILY.v4;
    },
    addressFamilyText() {
      return this.addressFamily ? "IPv4" : "IPv6";
    },
    showGraphs() {
      return this.loadingStatus == LOADING_STATUS.LOADED;
    },
    headerString() {
      switch (this.loadingStatus) {
        case LOADING_STATUS.LOADING:
          return this.$t("Networks.headerString.loading");
        case LOADING_STATUS.NOT_FOUND:
          return this.$t("Networks.headerString.notFound");
        case LOADING_STATUS.EXPIRED:
          return this.$t("Networks.headerString.expired");
        case LOADING_STATUS.LOADED:
          return isoCountries[this.countryCode];
        default:
        case LOADING_STATUS.ERROR:
          return this.$t("genericErrors.ups");
      }
    },
    subHeader() {
      switch (this.loadingStatus) {
        case LOADING_STATUS.LOADING:
          return this.$t("Networks.subHeader.loading");
        case LOADING_STATUS.NOT_FOUND:
          return this.$t("Networks.subHeader.notFound");
        case LOADING_STATUS.EXPIRED:
          return this.$t("Networks.subHeader.expired");
        case LOADING_STATUS.LOADED:
          return this.countryCode;
        default:
        case LOADING_STATUS.ERROR:
          return this.$t("genericErrors.badHappened");
      }
    }
  },
  watch: {
    addressFamily() {
      this.pushRoute();
    },
  }
};
</script>

<style lang="stylus">
@import '../styles/quasar.variables';
</style>
