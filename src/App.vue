<script setup lang="ts">
import { ref } from "vue";
import { State, assign, createMachine, interpret } from "xstate";

const message = ref([]);
const mockWsConnect = () => {
  return new Promise((resolve, reject) => {
    const isSuccess = Math.random() > 0.2;
    if (isSuccess) {
      const timer = setInterval(() => {
        // assign({
        //   message: [...context.message, { drugCode: 101 }],
        // });
        if (optionState.value.matches("scanning")) {
          message.value.push({ drugCode: 101 });
        }
        // console.log("接收ws");
        // context.message.push();
      }, 300);
      resolve({ close: () => clearInterval(timer) });
      // setInterval()
    } else {
      reject("ws失败");
    }
  });
};

const startScanApi = () => {
  return new Promise((resolve, reject) => {
    const isSuccess = Math.random() > 0.2;
    if (isSuccess) {
      resolve("start成功");
    } else {
      reject("start失败");
    }
  });
};
const continueScanApi = () => {
  return new Promise((resolve, reject) => {
    const isSuccess = Math.random() > 0.2;
    if (isSuccess) {
      resolve("continue成功");
    } else {
      reject("continue失败");
    }
  });
};

const clearScanApi = (type: any) => {
  return new Promise((resolve, reject) => {
    const isSuccess = Math.random() > 0.2;
    if (isSuccess) {
      resolve({ type, mes: "clear成功" });
    } else {
      reject("clear失败");
    }
  });
};

const stopScanApi = () => {
  return new Promise((resolve, reject) => {
    const randomNumber = Math.random();
    const isSuccess = randomNumber > 0.2;
    if (isSuccess) {
      if (randomNumber > 0.8) {
        resolve({
          mes: "没有扫到任何标签",
          data: [],
        });
      } else if (randomNumber > 0.4) {
        resolve({
          mes: "配置通过",
          data: ["aaa"],
        });
      } else {
        resolve({
          mes: "配置不通过",
          data: ["aaa", { drugCode: 111 }],
        });
      }
    } else {
      reject("start失败");
    }
  });
};
const getContext = () => ({
  ws: null,
  isDialogOpen: false,
  scanResult: [],
});

const optionMachine = createMachine(
  {
    predictableActionArguments: true,
    id: "optionMachine",
    initial: "unready",
    context: getContext(),
    states: {
      // 未就绪
      unready: {
        type: "compound",
        initial: "idle",
        states: {
          idle: {
            on: {
              open: {
                target: "tryconnnect",
              },
            },
          },
          tryconnnect: {
            entry: () => {
              console.log("尝试连接");
            },
            always: [
              {
                target: "wsconnecting",
                actions: () => {
                  console.log("开始连接");
                },
              },
            ],
          },
          wsconnecting: {
            invoke: {
              src: mockWsConnect,
              onDone: {
                target: "wsconnected",
                actions: "setWs",
              },
              onError: {
                actions: () => {
                  console.log("连接失败");
                },
                target: "idle",
              },
            },
          },
          wsconnected: {
            always: { target: "#optionMachine.ready" },
          },
        },
      },
      // 就绪
      ready: {
        initial: "idle",
        entry: [
          () => {
            console.log("就绪状态准备开始");
          },
          "openDialog",
        ],
        states: {
          idle: {
            on: {
              start: { target: "tryconnnect" },
              end: [
                // {
                //   target: "#optionMachine.unready",
                //   actions: (context) => {
                //     // context.ws.close();
                //     console.log("关闭ws");
                //   },
                //   cond: "isWsExist",
                // },
                {
                  target: "#optionMachine.unready",
                  actions: ["closeWs", "closeDialog"],
                },
              ],
            },
          },
          tryconnnect: {
            always: {
              target: "starting",
            },
          },
          starting: {
            invoke: {
              id: "tryStartApi",
              src: startScanApi,
              onDone: {
                target: "#optionMachine.scanning",
                actions: [
                  () => {
                    console.log("start成功");
                  },
                ],
              },
              onError: {
                target: "idle",
                actions: [
                  () => {
                    console.log("start失败请重试");
                  },
                ],
              },
            },
          },
        },
      },
      // 扫描中
      scanning: {
        entry: () => {
          console.log("正在扫描");
        },
        on: {
          stop: { target: "#optionMachine.stop" },
        },
      },
      // 暂停并检查
      stop: {
        invoke: {
          src: stopScanApi,
          onDone: [
            {
              target: "#optionMachine.showResult.full",
              actions: () => {
                console.log("提示扫描成功");
              },
              cond: "isResultFull",
            },
            {
              target: "#optionMachine.showResult.short",
              actions: "setShortResult",
              // cond: "isResultShort",
              // }, {
              // target:'continue'
            },
          ],
          onError: {
            target: "scanning",
            actions: () => {
              console.log("stop失败，请重试");
            },
          },
        },
      },
      continue: {
        entry: () => {
          console.log("物品缺失");
        },
        initial: "idle",
        states: {
          waitforcontinue: {},
          idle: {
            on: {
              continue: {
                target: "trycontinue",
              },
              next: {
                target: "#optionMachine.clear",
              },
            },
          },
          trycontinue: {
            invoke: {
              src: continueScanApi,
              onError: {
                target: "idle",
                actions: () => {
                  console.log("continue失败");
                },
              },
              onDone: {
                target: "#optionMachine.scanning",
                actions: () => {
                  console.log("continue成功");
                },
              },
            },
          },
        },
      },
      hist: {
        type: "history",
        history: "deep",
      },
      showResult: {
        states: {
          short: {
            on: {
              next: {
                target: "#optionMachine.continue",
              },
            },
          },
          full: {
            on: {
              next: {
                target: "#optionMachine.clear",
              },
            },
          },
        },
      },
      end: {},
      clear: {
        initial: "tryclear",
        entry: "clearMessage",
        states: {
          waitforclear: {
            on: {
              clear: {
                target: "tryclear",
              },
            },
          },
          tryclear: {
            invoke: {
              src: (context, event) => {
                if (context.scanResult.length > 0) {
                  return clearScanApi(2);
                } else {
                  // TODO 根据并行出入库设置0或者1
                  return clearScanApi(0);
                }
              },
              onDone: {
                // TODO 没什么要设置的吧
                actions: assign((context) => {
                  console.log("clear成功");
                  return {};
                }),
                target: "#optionMachine.ready",
              },
              onError: {
                actions: () => {
                  console.log("clear失败，请重试");
                },
                target: "waitforclear",
                // target: "hist",
              },
            },
          },
        },
      },
    },
  },
  {
    actions: {
      closeWs: assign((context) => {
        if (context.ws) {
          context.ws.close();
          return { ws: null };
        }
        return {};
      }),
      clearMessage: (context) => {
        if (message.value.length) {
          message.value.length = 0;
          // context.message = 0;
          // return { message: [] };
        }
        // return {};
      },
      openDialog: assign({ isDialogOpen: true }),
      closeDialog: assign({ isDialogOpen: false }),
      setWs: assign({
        ws: (context, event) => event.data,
      }),
      setShortResult: assign({
        scanResult: (context, event) => {
          return event.data.data || [];
        },
      }),
    },
    guards: {
      isWsExist: (context) => {
        if (context.ws) {
          return true;
        } else {
          return false;
        }
      },
      isResultFull: (context, event) => {
        console.log(event.data);
        return event.data.data.length === 1;
      },
      isResultEmpty: (context, event) => {
        // 返回的{ data: null } ，或者{data:[]}
        if (!event.data.data) return true;
        return event.data.data.length === 0;
      },
      isResultShort: (context, event) => {
        return event.data.data.length > 1;
      },
    },
  }
);

const stateRecord = ref<Array<State>>([]);
// const { state, send }=useMachine(optionMachine);
const optionState = ref<any>(getContext());
const optionServie = interpret(optionMachine)
  .onTransition((state) => {
    optionState.value = state;
    stateRecord.value.push(state.value);
  })
  .start();
</script>

<template>
  <div>
    <!-- {{ optionState.context }} -->
    <button @click="optionServie.send('open')">开始入库</button>
    <div
      class=""
      v-if="optionState.matches({ showResult: 'full' })"
      @click="optionServie.send('next')"
    >
      扫描成功提示
    </div>
    <div
      class=""
      v-if="optionState.matches({ showResult: 'short' })"
      @click="optionServie.send('next')"
    >
      扫描缺失结果
    </div>
    <div class="option-dialog" v-if="optionState.context.isDialogOpen">
      <!-- {{ optionState.context }} -->
      <!-- 会变长是因为存在history，不是bug -->
      {{ optionState.value }}
      <!-- {{ optionState.context.scanResult }} -->
      <button
        @click="optionServie.send('start')"
        v-if="optionState.matches({ ready: 'idle' })"
      >
        开始
        <!-- {{ optionState.context.buttonText1 }} -->
      </button>
      <button
        @click="optionServie.send('stop')"
        v-if="optionState.matches('scanning')"
      >
        暂停并检查
      </button>
      <button
        @click="optionServie.send('continue')"
        v-if="optionState.matches({ continue: 'idle' })"
      >
        重新扫描
      </button>
      <!-- <button @click="optionServie.send('')">结束</button> -->
      <button
        @click="optionServie.send('clear')"
        v-if="optionState.matches({ clear: 'waitforclear' })"
      >
        清除
      </button>
      <button
        @click="optionServie.send('next')"
        v-if="optionState.matches({ continue: 'idle' })"
      >
        下一个
      </button>
      <button
        @click="optionServie.send('end')"
        v-if="optionState.matches({ ready: 'idle' })"
      >
        结束
      </button>
      {{ message }}
    </div>
    {{ stateRecord }}
  </div>
</template>

<style scoped>
.option-dialog {
  box-shadow: 4px 4px 10px gray;
  height: 400px;
  width: 200px;
  position: absolute;
  /* left: 50%; */
  top: 10%;
  /* transform: translate(-50%, -50%); */
}
</style>
