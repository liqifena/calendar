<template>
  <div class="comp-wrap">
    <div class="ctx-box">
      <ul class="week-box">
        <li class="item-box" v-for="(item, i) in cNavs" :key="i">{{ item }}</li>
      </ul>
      <div class="list-box" ref="boxRef">

        <div class="item-box" :class="{ invalid: !item.flag }" v-for="(item, i) in list" :key="i">
          <div class="item-box-content">
            <div>
              <div class="date-box" :class="{ active: isMark && item.flag && item.date === currD }">
                <div class="text">{{ item.text }}</div>
                <div class="lunar" v-if="isLunar">{{ item.lunar }}</div>
              </div>
            </div>
            <div class="todo-box">
              <template v-for="(item1, i1) in item.todo" :key="i1">
                <div class="todo" :title="item1.description" :class="['color_' + item1.type]"
                     @click.stop.prevent="handleView(item1)">{{
                    item1.description
                  }}
                </div>
              </template>
            </div>

          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { toRefs, reactive, computed, onMounted, watch, watchEffect } from "vue";
import calendar from "./calendar.js";
import dayjs from "dayjs";
export default {
  name: "Calendar",
  props: {
    // 是否启用西方模式
    isWest: {
      type: Boolean,
      default: false,
    },
    // 是否开启农历
    isLunar: {
      type: Boolean,
      default: true,
    },
    // 是否标注今天
    isMark: {
      type: Boolean,
      default: true,
    },
    value: {
      type: String,
      default: '',
    },
    data: {
      type: Array,
      default: () => [
        {
          date: '2025-07-01',
          list: [
            {
              description: 'xxxxx',
              type: '1'
            },
            {
              description: 'xxxxx',
              type: '2'
            },
          ]
        }
      ],
    },
  },
  emits: ['click'],
  setup(props, { emit }) {
    let vm = reactive({
      boxRef: null,
      list: [],
      currYM: "",
      prevYM: "",
      nextYM: "",
      currD: dayjs().format("YYYY-MM-DD"),
      obj: {},
      css: {},
    });

    vm.cNavs = computed(() => {
      let arr = [..."一二三四五六日"];
      props.isWest && arr.unshift(arr.pop());
      return arr.map((item) => "周" + item);
    });

    vm.handleDateMark = (strDate = dayjs().format("YYYY-MM"), isWest = false) => {
      let { zh, en, zhPad, enPad } = [
        { zh: "日", en: "Sun", zhPad: 6, enPad: 0 },
        { zh: "一", en: "Mon", zhPad: 0, enPad: 1 },
        { zh: "二", en: "Tue", zhPad: 1, enPad: 2 },
        { zh: "三", en: "Wed", zhPad: 2, enPad: 3 },
        { zh: "四", en: "Thu", zhPad: 3, enPad: 4 },
        { zh: "五", en: "Fri", zhPad: 4, enPad: 5 },
        { zh: "六", en: "Sat", zhPad: 5, enPad: 6 },
      ][new Date(strDate).getDay()];
      let [year, month] = strDate.split("-");
      let cDays = new Date(+year, +month, 0).getDate();

      // 上个月多少天
      let [pY, pM, pDays] = strDate.split("-");
      pM = --pM ? +pM : 12;
      pY = pM !== 12 ? +pY : --pY;
      pDays = new Date(pY, pM, 0).getDate();

      // 下个月多少天
      let [nY, nM, nDays] = strDate.split("-");
      nM = ++nM !== 13 ? +nM : 1;
      nY = nM !== 1 ? +nY : ++nY;
      nDays = new Date(nY, nM, 0).getDate();

      let pPad = isWest ? enPad : zhPad;
      let nPad = 42 - cDays - pPad;

      let pDate = [pY, String(pM).padStart(2, 0)].join("-");
      let cDate = [year, String(month).padStart(2, 0)].join("-");
      let nDate = [nY, String(nM).padStart(2, 0)].join("-");

      return {
        zh,
        en,
        pPad,
        nPad,
        pDays,
        cDays,
        nDays,
        pDate,
        cDate,
        nDate,
        isWest,
        cDay: String(new Date().getDate()).padStart(2, 0),
        cList: [pPad, cDays, nPad].reduce((prev, item, index) => {
          let date = [pDate, cDate, nDate][index];
          let temp = [];
          for (let i = 0; i < item; i++) {
            let text = !index ? pDays + i + 1 - item : i + 1;
            temp.push({ text, date: [date, String(text).padStart(2, 0)].join("-") });
          }
          return prev.concat([temp]);
        }, []),
      };
    };
    vm.handleInit = (strDate = dayjs().format("YYYY-MM")) => {
      let { cDate, pDate, nDate, cDay, cList } = vm.handleDateMark(strDate, props.isWest);
      vm.currYM = cDate.replace(/(\d+)-(\d+)/g, "$1年$2月");
      vm.prevYM = pDate;
      vm.nextYM = nDate;
      // vm.currD = +cDay;
      vm.list = cList.flat().map((item) => {
        const lunarDate = calendar.solar2lunar(...item.date.match(/\d+/g))
        let { IMonthCn, IDayCn } = lunarDate;
        item.lunar = IDayCn == '初一' ? IMonthCn : IDayCn;
        // 判断是否是当前月
        item.flag = item.date.includes(cDate);
        return item;
      });
      vm.obj = {};
    };
    vm.handleView = (todo) => {
      console.log(todo,'toto');
      emit('click', todo);
    };

    onMounted(() => {
      if (props.value) {
        vm.handleInit(props.value);
      } else {
        vm.handleInit();
      }
    });
    watchEffect(() => {
      if (props.data?.length && vm.list.length) {
        vm.list = vm.list.map(item => {
          if (item.flag) {
            const data = props.data.find(item1 => {
              const day = item1.date.split("-")[2];
              return item.text == day;
            });
            if (data) {
              item.todo = data.list;
            }
          }
          return item;
        });
      }
    });
    watch(() => props.value, (newVal) => {
      if (newVal) {
        vm.handleInit(newVal);
      }
    });

    return { ...toRefs(vm) };
  },
};
</script>

<style lang="less" scoped>
li, ul {
  list-style: none;
}
.comp-wrap {
  width: 100%;
  height: 100%;

  .ctx-box {
    font-size: 14px;
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;

    .item-box {
      color: #6f6f6f;
      box-sizing: border-box;
    }

    .week-box {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      justify-items: start;
      font-size: 16px;
      background: #F0F2F4;
      box-shadow: 0px 1px 0px 0px rgba(0, 0, 0, 0.08), 0px -1px 0px 0px rgba(0, 0, 0, 0.08), 1px 0px 0px 0px rgba(0, 0, 0, 0.08), -1px 0px 0px 0px rgba(0, 0, 0, 0.08);

      .item-box {
        position: relative;
        height: 32px;
        line-height: 32px;
        color: rgba(0, 0, 0, 0.88);
        padding-left: 18px;
      }
    }

    .list-box {
      flex: 1;
      position: relative;
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      grid-template-rows: repeat(6, 1fr);
      align-items: start;
      min-height: 0;

      &::after {
        content: "";
        position: absolute;
        right: 0;
        top: 0;
        bottom: 0;
        color: #e8ecef;
        border-left: 1px solid;
        pointer-events: none;
      }

      .item-box {
        position: relative;
        overflow: hidden;
        padding: 12px;
        width: 100%;
        height: 100%;
        box-sizing: border-box;
        color: #0A1018;

        .item-box-content {
          display: flex;
          flex-direction: column;
          width: 100%;
          height: 100%;
          min-height: 0;
        }

        .date-box {
          display: inline-flex;
          gap: 12px;
          padding: 2px 6px;
          border-radius: 4px;

          &.active {
            background: linear-gradient(73deg, #8976E9 0%, #4A4AAE 100%);
            color: #fff;
          }
        }

        &.invalid {
          color: #A0A0A0;
        }

        &::after {
          content: "";
          position: absolute;
          left: 0;
          top: 0;
          right: 0;
          bottom: 0;
          color: #e8ecef;
          border-left: 1px solid;
          border-bottom: 1px solid;
          pointer-events: none;
        }

        .lunar {
          font-size: 12px;
        }

        .todo-box {
          flex: 1;
          display: flex;
          flex-direction: column;
          gap: 12px;
          margin-top: 12px;
          max-height: calc(100% - 30px);
          overflow-y: auto;

          &::-webkit-scrollbar-track-piece {
            background-color: #f8f8f8;
          }

          &::-webkit-scrollbar {
            width: 7px;
            height: 7px;
          }

          &::-webkit-scrollbar-thumb {
            background-color: #e5e5e5;
            background-clip: padding-box;
          }

          &::-webkit-scrollbar-thumb:hover {
            background-color: #bbb;
          }
        }

        .todo {
          position: relative;
          padding: 2px 12px;
          font-size: 12px;
          color: #0A1018;
          cursor: pointer;
          height: 22px;
          background: rgba(62, 198, 110, 0.2);
          border-radius: 3px 4px 4px 3px;
          flex: none;
          white-space: nowrap;
          text-overflow: ellipsis;
          overflow: hidden;

          &::before {
            content: '';
            position: absolute;
            left: 0;
            top: 0;
            width: 4px;
            height: 100%;
            background: #3EC66E;
            border-radius: 11px;
          }

          &.color_1 {
            background: rgba(62, 198, 110, 0.2);

            &::before {
              background: #3EC66E;
            }
          }

          &.color_2 {
            background: rgba(255, 174, 66, 0.2);

            &::before {
              background: #FFAE42;
            }
          }

          &.color_3 {
            background: rgba(111, 198, 231, 0.2);

            &::before {
              background: #6FC6E7;
            }
          }

        }
      }


    }
  }
}
</style>