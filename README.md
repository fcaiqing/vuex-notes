### vuex学习笔记
- [模块解析方式](#模块解析方式)
- [响应式原理](#响应式原理)
- [模块安装](#模块安装)
* #### 模块解析方式
>通过深度优先进行节点遍历，每遍历节点时更新路径进行路径跟踪，从而获得目标数据

1. 如下就是通过路径跟踪获取当前节点的父节点
```javascript
const parent = path.slice(0, -1).reduce((node, k) => {
    return node.children[k]
}, root)
```
2. 过程示例代码如下
```javascript
const options = {
    modules: {
        a: {
            namespaced: true,
            state: {
                count: 0
            },
            modules: {
                aa: {
                    state: {
                        subCount: 0
                    }
                }
            }
        },
        b: {
            namespaced: true,
            state: {
                age: 0
            },
            modules: {
                bb: {
                    namespaced: true,
                    state: {
                        subAge: 90
                    }
                }
            }
        }
    }
}

var root = null

class Module {
    constructor(obj) {
        this.state = obj.state
        this.children = Object.create(null)
    }
}

/**
 * @description: 
 * @param {array} path 节点路径
 * @param {object} obj 节点
 * @return: 
 */
function registerModule(path, obj) {
    const module = new Module(obj)
    if (path.length == 0) {
        root = new Module(obj)
    } else {
        const parent = path.slice(0, -1).reduce((node, k) => {
            return node.children[k]
        }, root)
        parent.children[path[path.length-1]] = module
    }
    if (obj.modules) {
        Object.keys(obj.modules).forEach(key => {
            registerModule(path.concat(key), obj.modules[key])
        })
    }
}
registerModule([], options)
//输出
{
    "children": {
        "a": {
            "state": {
                "count": 0
            },
            "children": {
                "aa": {
                    "state": {
                        "subCount": 0
                    },
                    "children": {}
                }
            }
        },
        "b": {
            "state": {
                "age": 0
            },
            "children": {
                "bb": {
                    "state": {
                        "subAge": 90
                    },
                    "children": {}
                }
            }
        }
    }
}
```
* ### 响应式原理
>通过定义属性存取器实现数据响应式
```javascript

function callback(k,val) {
    console.log(`${k} data updated to ${val}`)
}

var obj = {
    title: "people list",
    people: [
        {
            name: "ca",
            age: 29,
            habits: ["music", "hiking"]
        },
        {
            name: "cq",
            age: 30,
            habits: ["music", "hiking"]
        },
        {
            name: "cv",
            age: 90,
            habits: ["music", "hiking"]
        },
    ]
}
/**
 * @description: 数据响应式化
 * @param {obj} 目标数据 
 * @return: void
 */
function activeDefine(obj) {
    if (typeof obj != "object") {
        throw new Error("param is not a object")
    }
    Object.keys(obj).forEach(k => {
        const value = obj[k]
        if (typeof obj[k] != "object") {
            Object.defineProperty(obj, k, {
                configurable: true,
                enumerable: true,
                get: () => {
                    return value
                },
                set: val => {
                    if (val === value) return
                    callback(k, val)
                }
            })
        } else {
            activeDefine(obj[k])
        }
    });
}
activeDefine(obj)
obj.title = "op"    //title data updated to op
obj.people[1].age = 189 //age data updated to 189

```
* ### 模块安装
>store各个模块的安装过程其实就是数据扁平化处理过程，处理过程中根据namespaced来决定是否启用命名空间
1. 实例代码
```javascript
var store = {
    state: {
        name: 'xxx'
    },
    getters: {
        name: () => {}
    },
    actions: {
        add: () => {}
    },
    mutations: {
        ADD_NAME: (state, payload) => {
            state.name += payload.last
        }
    },
    modules: {
        a: {
            state: {
                name: 'xxx'
            },
            getters: {
                name: () => {}
            },
            actions: {
                add: () => {}
            },
            mutations: {
                ADD_NAME: (state, payload) => {
                    state.name += payload.last
                }
            },
            modules: {
                aa: {
                    state: {
                        name: 'xxx'
                    },
                    getters: {
                        name: () => {}
                    },
                    actions: {
                        add: () => {}
                    },
                    mutations: {
                        ADD_NAME: (state, payload) => {
                            state.name += payload.last
                        }
                    }
                }
            }
        },
        b: {
            state: {
                name: 'xxx'
            },
            getters: {
                name: () => {}
            },
            actions: {
                add: () => {}
            },
            mutations: {
                ADD_NAME: (state, payload) => {
                    state.name += payload.last
                }
            }
        }
    }
}

/**
 * description: 数据扁平化
 * @param {object} store 根模块
 * @param {object} rootState 根state
 * @param {array} path 子模块路径
 * @param {object} module 子模块
 * return: {object} 返回新的根模块
 */
function installModule(store, rootState, path, module) {
    if (path.length) {
        let parentState = getParentState(rootState, path.slice(0, -1))
        let namespaced = getNamespaced(path)
        parentState[path[path.length-1]] = module.state
        store.modules[namespaced] = module
        forEach(module.getters, (handle, key) => {
            store.getters[namespaced + key] = handle
        })
        forEach(module.actions, (handle, key) => {
            store.actions[namespaced + key] = handle
        })
        forEach(module.mutations, (handle, key) => {
            store.mutations[namespaced + key] = handle
        })
    }
    if (module.modules) {
        forEach(module.modules, (module, key) => {
            installModule(store, rootState, path.concat(key), module)
        })
    }
}
installModule(store, store.state, [], store)
console.log(store)
/**
 * 
 * @param {object} rootState root state
 * @param {array} path 模块路径
 * return：{object} 当前模块父state
 */
function getParentState(rootState, path) {
    return path.reduce((state, key) => {
        return state[key]
    }, rootState)
}
/**
 * 获取路径对应的命名空间
 * @param {array} path 模块路径
 * return: {string}
 */
function getNamespaced(path) {
    return path.reduce((namespaced, key) => {
        return namespaced + (key + "/")
    }, "")
}
/**
 * 遍历对象
 * @param {object} obj 
 * @param {function} fn  
 * return: void
 */
function forEach(obj, fn) {
    Object.keys(obj).forEach(key => {
        fn(obj[key], key)
    })
}

```
