# VUEX 在 VUE 3 中的使用

vuex 是基于vue框架的一个状态管理库。可以管理复杂应用的数据状态，比如兄弟组件的通信、多层嵌套的组件的传值等等。

vuex 核心概念：

- State: 提供唯一的公共数据源，所有共享的数据都要统一放到 Store 的 State 中进行存储。
- Getter: 用于对 Store 中的数据进行加工处理形成新的数据。
  - Getter 可以对 Store 中已有的数据加工处理之后形成新的数据，类似 Vue 的计算属性。
  - Store 中数据发生变化，Getter 的数据也会跟着变化。
- Mutation：用于变更 Store中 的数据。
  - 只能通过 mutation 变更 Store 数据，不可以直接操作 Store 中的数据。
  - 通过这种方式虽然操作起来稍微繁琐一些，但是可以集中监控所有数据的变化。
- Action：用于处理异步任务。
  >如果通过异步操作变更数据，必须通过 Action，而不能使用 Mutation，但是在 Action 中还是要通过触发 Mutation 的方式间接变更数据。
- Module: 将store分割成模块,每个模块都有state、mutation、action、getter、甚至嵌套子模块

## 安装和配置

### 安装

```bash
npm install vuex
```

### 配置

1. 在src目录下新建store目录
2. 在store目录下新建index.ts文件
3. 在index.ts 中编写如下代码

    ```javascript
    // 导入vuex模块
    import { createStore } from "vuex";
    
    // 定义一个存储数据的结构 --相当于实体类
    
    export interface MyState {
        collapse: boolean,  // 控制侧边栏收缩
    }
    
    // 创建一个store对象
    
    export const store = createStore<MyState>({
        // state -- 存储具体的值
        state: {
            collapse: false,
        },
        // mutations -- 修改state中的值
        mutations:{
            // 修改collapse的值
            setCollapse(state: MyState, collapse:boolean){
                state.collapse = collapse
            }
        },
        //getter -- 获取state中的值
        getters:{
            getCollapse(state: MyState){
                return state.collapse;
            }
        }
    })
    ```

4. 在main.ts文件中注册

    ```javascript
    import {store} from './store' //导入创建的store对象
    app.use(store)
    ```

5. 使用

    ```javascript
    import { useStore } from "vuex";
    // 获取当前vuex中的store对象
    const store = useStore();
    // 定义响应式变量
    const localCollaspe = ref(false)
    // 获取collaspe的值
    const isCollaspe = computed(()=>{
        localCollaspe.value = store.getters['getCollapse']
        return store.getters['getCollapse']
    })
    
    // 定义函数实现修改collapse的值
    const changeCollapse = () => {
        // 取反
        localCollaspe.value = ! localCollaspe.value
        store.commit('setCollapse', localCollaspe.value)
    }
    ```
