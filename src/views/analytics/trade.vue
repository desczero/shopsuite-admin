<template>
  <div>
    <div class="dashboard-container">
      <el-row :gutter="24">
        <el-col :lg="24" :md="24" :sm="24" :xl="24" :xs="24">
          <ms-search-box style="float: right;">
            <ms-search-box-right-panel :span="24">
              <el-form :inline="true" :model="queryForm" @submit.prevent>
                <el-form-item class="ml10">
                  <ms-date-range-picker v-model="order_time_range"/>
                </el-form-item>
                <el-form-item>
                  <el-button :icon="Search" type="primary" @click="fetchData">
                    {{ t('查询') }}
                  </el-button>
                </el-form-item>
              </el-form>
            </ms-search-box-right-panel>
          </ms-search-box>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :lg="6" :md="12" :sm="24" :xl="6" :xs="24">
          <top-num
            background="white"
            :bottom-text="t('订单金额')"
            :count-config="orderAmountConfig"
            icon="product-hunt-line"
            :title="t('订单金额')"
            :unit-text="t('个')"
          />
        </el-col>
        <el-col :lg="6" :md="12" :sm="24" :xl="6" :xs="24">
          <top-num
            background="white"
            :bottom-text="t('订单数量')"
            :count-config="orderNumConfig"
            icon="user-search-line"
            :title="t('订单数量')"
            :unit-text="t('单')"
          />
        </el-col>
        <el-col :lg="6" :md="12" :sm="24" :xl="6" :xs="24">
          <top-num
            background="white"
            :bottom-text="t('退款金额')"
            :count-config="refundAmountConfig"
            icon="user-3-line"
            :title="t('退款金额')"
            :unit-text="t('人')"
          />
        </el-col>
        <el-col :lg="6" :md="12" :sm="24" :xl="6" :xs="24">
          <top-num
            background="white"
            :bottom-text="t('退款单数')"
            :count-config="refundNumConfig"
            icon="database-2-line"
            :title="t('退款单数')"
            :unit-text="t('个')"
          />
        </el-col>
      </el-row>

    </div>


    <el-row :gutter="24">
      <el-col :span="24">
        <div class="grid-content bg-purple">
          <el-card v-loading="loading" class="box-card">
            <template #header>
              {{ t('订单销售金额对比图') }}
              <el-radio-group v-model="saleTime" size="small" style="float: right;">
                <el-radio-button :label="t('7天')" @click="getSaleOrderAmountFun(7)" />
                <el-radio-button :label="t('30天')" @click="getSaleOrderAmountFun(30)" />
                <el-radio-button :label="t('3个月')" @click="getSaleOrderAmountFun(90)" />
                <el-radio-button :label="t('半年')" @click="getSaleOrderAmountFun(180)" />
              </el-radio-group>
            </template>
            <ms-chart
              :init-options="initOptions"
              :option="polylineOptionMake"
              style="width: 100%"
              theme="ms-echarts-theme"
            />
          </el-card>
        </div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import {translate as t} from '@/i18n'
import TopNum from '@/plugins/MsWidget/TopNum'
import {
  getSaleOrderAmount,
  getOrderAmount,
  getOrderNum,
  getOrderNumTimeline
} from '@/api/analytics/order'

import {
  getReturnAmount,
  getReturnNum,
  getReturnAmountTimeline,
  getReturnNumTimeline
} from '@/api/analytics/return'

import MsChart from '@/plugins/MsChart'

import {useSettingsStore} from '@/store/modules/settings'
import moment from "dayjs";
import {formatTimeLine, formatTimeLineRange} from "@/utils";
import {getList as getUserList} from "@/api/account/userInfo";

export default defineComponent({
  name: 'AnalyticsTrade',
  components: {MsChart, TopNum},
  setup() {
    const $tableHeight = inject('$tableHeight')

    const settingsStore = useSettingsStore()
    const {echartsGraphic2} = storeToRefs(settingsStore)
    const state = reactive({
      saleTime: '30天',
      orderAmountConfig: {
        startVal: 0,
        endVal: 0,
        decimals: 2,
        prefix: '',
        suffix: '',
        separator: ',',
        duration: 8000,
      },
      refundAmountConfig: {
        startVal: 0,
        endVal: 0,
        decimals: 0,
        prefix: '',
        suffix: '',
        separator: ',',
        duration: 8000,
      },
      orderNumConfig: {
        startVal: 0,
        endVal: 0,
        decimals: 0,
        prefix: '',
        suffix: '',
        separator: ',',
        duration: 8000,
      },
      refundNumConfig: {
        startVal: 0,
        endVal: 0,
        decimals: 0,
        prefix: '',
        suffix: '',
        separator: ',',
        duration: 8000,
      },
      form: {
        startVal: 0,
        decimals: 0,
        prefix: '',
        suffix: '',
        separator: ',',
        duration: 5000,
      },
      data: {
        order_num: 0,
        favorable_order_num: 0,
        user_reg_yestoday: [
          {
            num: 0,
          },
          {
            num: 0,
          },
        ],
        user_payment_yestoday: [
          {
            num: 0,
          },
          {
            num: 0,
          },
        ],
      },
      polylineOptionMake: {
        /*title: {
          left: 'center',
          text: t('最近7天的订单成交金额趋势图'),
        },*/
        tooltip: {
          trigger: 'axis',
        },
        legend: {
          data: ['订单数量', '成交金额', '退单数量', '退单金额'],
        },
        grid: {
          right: '4%',
          bottom: '3%',
          containLabel: true,
        },
        xAxis: {
          type: 'category',
          boundaryGap: false,
          data: [],
        },
        yAxis: {
          type: 'value',
        },
        series: [
          {
            name: t('订单数量'),
            type: 'line',
            data: [],
          },
          {
            name: t('成交金额'),
            type: 'line',
            data: [],
          },
          {
            name: t('退单数量'),
            type: 'line',
            data: [],
          },
          {
            name: t('退单金额'),
            type: 'line',
            data: [],
          },
        ],
      },
      columnLoading: false,
      polylineLoading: false,
      loading: false,
      initOptions: {
        renderer: 'svg',
      },


      height: $tableHeight(),
      itemList: [],
      listLoading: true,
      layout: 'total, sizes, prev, pager, next, jumper',
      total: 0,
      selectRows: '',

      order_time_range: ["", ""],
      userList: [],
      queryForm: {
        stime: "",
        etime: ""
      },
    })

    watch(
      () => echartsGraphic2.value,
      () => {
        state.option.series[0].itemStyle.color =
          new MsChart.graphic.LinearGradient(
            0,
            0,
            0,
            1,
            echartsGraphic2.value.map((color, offset) => ({
              color,
              offset,
            }))
          )
      }
    )


    const goToPath = (path, query) => {
      state.$router.push({path: path, query: query})
    }


    const queryData = () => {
      fetchData()
    }

    const fetchData = () => {

      //
      if (state.order_time_range.length > 0) {
        if (state.order_time_range[0]) {
          state.queryForm.stime = state.order_time_range[0].getTime();
        }

        if (state.order_time_range[1]) {
          state.queryForm.etime = state.order_time_range[1].getTime();
        }
      } else {
        state.queryForm.stime = null
        state.queryForm.etime = null
      }

      const params = state.queryForm

      getOrderAmountFun(params)
      getReturnAmountFun(params)
      getOrderNumFun(params)
      getReturnNumFun(params)

      let days = 30
      if (state.queryForm.stime && state.queryForm.etime) {
        state.saleTime = ''
        days = (state.queryForm.etime - state.queryForm.stime) / (1000 * 3600 * 24)
        const timeRange = {
          days : days,
          stime : state.queryForm.stime,
          etime : state.queryForm.etime
        }

        getSaleOrderAmountFunRange(timeRange);
      } else {
        getSaleOrderAmountFun(days)
      }
      //listOrderItemNumFun()
    }

    /**
     * 订单销售金额对比图范围
     * @param days
     * @returns {Promise<void>}
     */
    const getSaleOrderAmountFunRange = async (timeRange) => {
      state.loading = true

      //最近days天
      const params = {stime: timeRange.stime, etime: timeRange.etime}

      const { data } = await getSaleOrderAmount(params)
      formatTimeLineRange(data, timeRange.days, timeRange.etime);

      const { data : dataNum } = await getOrderNumTimeline(params)
      formatTimeLineRange(dataNum, timeRange.days, timeRange.etime);

      const { data : returnAmount } = await getReturnAmountTimeline(params)
      formatTimeLineRange(returnAmount, timeRange.days, timeRange.etime);

      const { data : returnNum } = await getReturnNumTimeline(params)
      formatTimeLineRange(returnNum, timeRange.days, timeRange.etime);

      //state.polylineOptionMake.title.text =  s('最近' + days + '天的订单成交金额趋势图')
      state.polylineOptionMake.xAxis.data = data.map((item) => item.time)
      state.polylineOptionMake.series[0].data = dataNum.map((item) => item.num)
      state.polylineOptionMake.series[1].data = data.map((item) => item.num)
      state.polylineOptionMake.series[2].data = returnNum.map((item) => item.num)
      state.polylineOptionMake.series[3].data = returnAmount.map((item) => item.num)
      state.loading = false
    }


    /**
     * 总额
     *
     * @returns {Promise<void>}
     */
    const getOrderAmountFun = async (params) => {
      const {data} = await getOrderAmount(params)
      data.today = data.current;
      data.yestoday = data.pre;
      state.orderAmountConfig = Object.assign(state.orderAmountConfig, data)
    }

    /**
     * 退款总额
     *
     * @returns {Promise<void>}
     */
    const getReturnAmountFun = async (params) => {
      const {data} = await getReturnAmount(params)
      data.today = data.current;
      data.yestoday = data.pre;
      state.refundAmountConfig = Object.assign(state.refundAmountConfig, data)
    }

    /**
     * 订单数
     *
     * @returns {Promise<void>}
     */
    const getOrderNumFun = async (params) => {
      const {data} = await getOrderNum(params)
      data.today = data.current;
      data.yestoday = data.pre;
      state.orderNumConfig = Object.assign(state.orderNumConfig, data)
    }

    /**
     * 退款单数
     *
     * @returns {Promise<void>}
     */
    const getReturnNumFun = async (params) => {
      const {data} = await getReturnNum(params)
      data.today = data.current;
      data.yestoday = data.pre;
      state.refundNumConfig = Object.assign(state.refundNumConfig, data)
    }


    /**
     * 订单销售金额对比图
     * @param days
     * @returns {Promise<void>}
     */
    const getSaleOrderAmountFun = async (days) => {
      state.loading = true

      //最近days天
      const start = new Date(moment(moment().subtract(days-1, "days").format("YYYY-MM-DD")));
      const end = new Date();
      const params = {stime:start.getTime(), etime:end.getTime()}

      const { data } = await getSaleOrderAmount(params)
      formatTimeLine(data, days);


      const { data : dataNum } = await getOrderNumTimeline(params)
      formatTimeLine(dataNum, days);

      const { data : returnAmount } = await getReturnAmountTimeline(params)
      formatTimeLine(returnAmount, days);

      const { data : returnNum } = await getReturnNumTimeline(params)
      formatTimeLine(returnNum, days);


      //state.polylineOptionMake.title.text =  s('最近' + days + '天的订单成交金额趋势图')
      state.polylineOptionMake.xAxis.data = data.map((item) => item.time)
      state.polylineOptionMake.series[0].data = dataNum.map((item) => item.num)
      state.polylineOptionMake.series[1].data = data.map((item) => item.num)
      state.polylineOptionMake.series[2].data = returnNum.map((item) => item.num)
      state.polylineOptionMake.series[3].data = returnAmount.map((item) => item.num)
      state.loading = false
    }

    const findRemoteUserList = (user_nickname) => {
      if (user_nickname !== '') {
        state.loading = true
        setTimeout(() => {
          state.loading = false
          getUserList({user_nickname: user_nickname}).then((res) => {
            state.userList = res.data.items
          })
        }, 200)
      } else {
        state.userList = []
      }
    }

    onMounted(() => {
      fetchData()
    })

    return {
      t,
      ...toRefs(state),
      close,
      goToPath,
      queryData,
      fetchData,
      getSaleOrderAmountFun,
      getSaleOrderAmountFunRange,
      getReturnAmountFun,
      getOrderNumFun,
      getReturnNumFun,
      findRemoteUserList
    }
  },
})
</script>
<style lang="scss" scoped>
.dashboard-container {
  padding: 0 !important;
  background: $base-color-background !important;

  :deep() {
    .el-card {
      height: 300px;

      [class*='-echart'] {
        width: 100%;
        height: 200px;
      }
    }
  }
}

.xe-icon {
  width: 80px;
  padding: 10px;

  .i-wh-workbench {
    display: block;
    color: #fff;
    text-align: center;
    font-size: 25px;
    line-height: 50px;
    width: 50px;
    height: 50px;
    border-radius: 50%;
    background-clip: padding-box;
  }
}

.xe-info {
}

.xe-num {
  display: block;
  font-size: 20px;
  font-weight: 400;
  color: #3e3f3f;
  text-align: left;
}

.xe-label {
  text-align: left;
  display: block;
  font-style: normal;
  font-size: 10px;
  text-transform: uppercase;
  color: #979898;
}

.xe-todo {
  text-align: right;
  display: block;
  font-style: normal;
  font-size: 10px;
  text-transform: uppercase;
  color: #979898;
  margin-top: 5px;
}

.el-row {
  margin-bottom: 0px;
}

:deep(.product-list) {
  .el-card__header {
    padding-bottom: 0 !important;
  }
}


.ml10 {
  margin-left: 10px;
}

</style>
