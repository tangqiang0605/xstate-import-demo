<script setup lang="ts">
import { ref } from "vue";
import { assign, createMachine, interpret, send } from "xstate";

const message = ref<Array<any>>([]);
const mockWsConnect = () => {
  return new Promise((resolve, reject) => {
    const isSuccess = Math.random() > 0.5;
    if (isSuccess) {
      const timer = setInterval(() => {
        if (optionState.value.matches({ step: "scanning" })) {
          message.value.push({ drugCode: 101 });
        }
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
    const isSuccess = Math.random() > 0.1;
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
    console.log(type);
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
const getContext = (): OptionMachineContext => ({
  ws: null,
  isDialogOpen: false,
  scanResult: [],
});

// 状态机处理的事件
type OptionMachineEvent = {
  type: string;
  data: any;
};

// 状态机的上下文（扩展状态）
type OptionMachineContext = {
  ws: WebSocket | null;
  isDialogOpen: boolean;
  scanResult: Array<any>;
};
const optionMachine = createMachine<OptionMachineContext, OptionMachineEvent>(
  {
    predictableActionArguments: true,
    id: "optionMachine",
    type: "parallel",
    context: getContext(),
    states: {
      // 平行状态机：出入库操作
      step: {
        initial: "unready",
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
                    actions: [
                      () => {
                        alert("WebSocket连接失败，请重试");
                        console.log("连接失败");
                      },
                    ],
                    target: "idle",
                  },
                },
              },
              wsconnected: {
                always: { target: "#optionMachine.step.ready" },
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
                    //   target: "#optionMachine.step.unready",
                    //   actions: (context) => {
                    //     // context.ws.close();
                    //     console.log("关闭ws");
                    //   },
                    //   cond: "isWsExist",
                    // },
                    {
                      target: "#optionMachine.step.unready",
                      actions: [
                        "closeWs",
                        "closeDialog",
                        send({ type: "end", to: "#optionMachine.method" }),
                      ],
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
                    target: "#optionMachine.step.scanning",
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
                        alert("开始失败，请重试");
                        // console.log("start失败请重试");
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
              stop: { target: "#optionMachine.step.stop" },
            },
          },
          // 暂停并检查
          stop: {
            invoke: {
              src: stopScanApi,
              onDone: [
                {
                  target: "#optionMachine.step.showResult.full",
                  actions: () => {
                    console.log("提示扫描成功");
                  },
                  cond: "isResultFull",
                },
                {
                  target: "#optionMachine.step.showResult.short",
                  actions: "setShortResult",
                  // cond: "isResultShort",
                  // }, {
                  // target:'continue'
                },
              ],
              onError: {
                target: "scanning",
                actions: () => {
                  alert("暂停失败，请重试");
                  // console.log("stop失败，请重试");
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
                    target: "#optionMachine.step.clear",
                  },
                },
              },
              trycontinue: {
                invoke: {
                  src: continueScanApi,
                  onError: {
                    target: "idle",
                    actions: () => {
                      alert("重新扫描失败，请重试");
                      console.log("continue失败");
                    },
                  },
                  onDone: {
                    target: "#optionMachine.step.scanning",
                    actions: () => {
                      console.log("continue成功");
                    },
                  },
                },
              },
            },
          },
          showResult: {
            states: {
              short: {
                on: {
                  next: {
                    target: "#optionMachine.step.continue",
                  },
                },
              },
              full: {
                on: {
                  next: {
                    target: "#optionMachine.step.clear",
                  },
                },
              },
            },
          },
          clear: {
            initial: "tryclear",
            exit: "clearMessage",
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
                  src: (context) => {
                    if (context.scanResult.length > 0) {
                      return clearScanApi(2);
                    } else {
                      // import 0 ,export 1
                      const type =
                        optionState.value.value.method == "import" ? 0 : 1;
                      return clearScanApi(type);
                    }
                  },
                  onDone: {
                    target: "#optionMachine.step.ready",
                  },
                  onError: {
                    actions: () => {
                      alert("清除失败，请重试");
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
      // 平行状态机：出入库类型
      method: {
        type: "compound",
        initial: "idle",
        states: {
          idle: {
            on: {
              import: {
                actions: send({
                  type: "open",
                  to: "#optionMachine.step.unready",
                }),
                target: "import",
              },
              export: {
                actions: send({
                  type: "open",
                  to: "#optionMachine.step.unready",
                }),
                target: "export",
              },
            },
          },
          import: {
            on: {
              end: { target: "idle" },
            },
          },
          export: {
            on: {
              end: { target: "idle" },
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
      clearMessage: () => {
        if (message.value.length) {
          message.value.length = 0;
        }
      },
      openDialog: assign({ isDialogOpen: true }),
      closeDialog: assign({ isDialogOpen: false }),
      setWs: assign({
        ws: (_, event) => event.data,
      }),
      setShortResult: assign({
        scanResult: (_, event) => {
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
      isResultFull: (_, event) => {
        console.log(event.data);
        return event.data.data.length === 1;
      },
      isResultEmpty: (_, event) => {
        // 返回的{ data: null } ，或者{data:[]}
        if (!event.data.data) return true;
        return event.data.data.length === 0;
      },
      isResultShort: (_, event) => {
        return event.data.data.length > 1;
      },
    },
  }
);

const stateRecord = ref<any>([]);
const optionState = ref<any>(getContext());
// const { state, send }=useMachine(optionMachine);
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
    <button @click="optionServie.send(['import'])">开始入库</button>
    <button @click="optionServie.send(['export'])">开始出库</button>
    <div
      class=""
      v-if="optionState.matches({ step: { showResult: 'full' } })"
      @click="optionServie.send('next')"
    >
      扫描成功提示
    </div>
    <div
      class=""
      v-if="optionState.matches({ step: { showResult: 'short' } })"
      @click="optionServie.send('next')"
    >
      扫描缺失结果
    </div>
    <div class="option-dialog" v-if="optionState.context.isDialogOpen">
      <!-- {{ optionState.context }} -->
      <!-- 会变长是因为存在history，不是bug -->
      <!-- {{ optionState.value }} -->
      <!-- {{ optionState.context.scanResult }} -->
      <button
        @click="optionServie.send('start')"
        v-if="optionState.matches({ step: { ready: 'idle' } })"
      >
        开始
        <!-- {{ optionState.context.buttonText1 }} -->
      </button>
      <button
        @click="optionServie.send('stop')"
        v-if="optionState.matches({ step: 'scanning' })"
      >
        暂停并检查
      </button>
      <button
        @click="optionServie.send('continue')"
        v-if="optionState.matches({ step: { continue: 'idle' } })"
      >
        重新扫描
      </button>
      <!-- <button @click="optionServie.send('')">结束</button> -->
      <button
        @click="optionServie.send('clear')"
        v-if="optionState.matches({ step: { clear: 'waitforclear' } })"
      >
        清除
      </button>
      <button
        @click="optionServie.send('next')"
        v-if="optionState.matches({ step: { continue: 'idle' } })"
      >
        下一个
      </button>
      <button
        @click="optionServie.send('end')"
        v-if="optionState.matches({ step: { ready: 'idle' } })"
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
  /* position: absolute; */
  /* left: 50%; */
  /* top: 10%; */
  /* transform: translate(-50%, -50%); */
}
</style>
