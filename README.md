### vuex学习笔记
- [模块解析方式](#模块解析方式)
- [响应式原理](#响应式原理)
- [模块安装](#模块安装)
- [依赖收集](#依赖收集)
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
>针对属性值为数组，通过重写数组方法通知更新
示例如下
```javascript
function defineArr(arr) {
    const arrfnc = ['push', 'splice']
    const _arr = Object.create(Array.prototype)
    arrfnc.forEach(func => {
        _arr[func] = function () {
            console.log("数据改变了", func)
            Array.prototype[func].apply(this, arguments)
        }
    })
    Object.setPrototypeOf(arr, _arr)
}
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
//输出
{
    state: {
        name: 'xxx',
        a: { name: 'xxx', aa: [Object] },
        b: { name: 'xxx' }
    },
    getters: {
        name: [Function: name],
        'a/name': [Function: name],
        'a/aa/name': [Function: name],
        'b/name': [Function: name]
    },
    actions: {
        add: [Function: add],
        'a/add': [Function: add],
        'a/aa/add': [Function: add],
        'b/add': [Function: add]
    },
    mutations: {
        ADD_NAME: [Function: ADD_NAME],
        'a/ADD_NAME': [Function: ADD_NAME],
        'a/aa/ADD_NAME': [Function: ADD_NAME],
        'b/ADD_NAME': [Function: ADD_NAME]
    },
    modules: {
        a: {
            state: [Object],
            getters: [Object],
            actions: [Object],
            mutations: [Object],
            modules: [Object]
        },
        b: {
            state: [Object],
            getters: [Object],
            actions: [Object],
            mutations: [Object]
        },
        'a/': {
            state: [Object],
            getters: [Object],
            actions: [Object],
            mutations: [Object],
            modules: [Object]
        },
        'a/aa/': {
            state: [Object],
            getters: [Object],
            actions: [Object],
            mutations: [Object]
        },
        'b/': {
            state: [Object],
            getters: [Object],
            actions: [Object],
            mutations: [Object]
        }
    }
}
```
* ### 依赖收集
>理由： 1. 只对用到的数据进行执行render function；2. 当属性变化时，能够通知所有的vue实例

1. 通过发布订阅模式实现，实例代码
```javascript
    //依赖收集服务，提供注册和通知更新服务
    class Dep {
        constructor() {
            this.sub = []
        }
        addSub(watcher, obj, key, value) {
            watcher.save(obj, key, value)
            this.sub.push(watcher)
        }
        notify(obj, k) {
            this.sub.forEach(watcher => {
                watcher.update(obj, k)
            })
        }
    }
    class Watcher {
        constructor(name) {
            this._obj = {}
            this._name = name
        }
        save(obj, key, value) {
            this._obj[key] = value
        }
        update(obj, k) {
            console.log(`${this._name}: 对象的${k}属性改变了，更新后的前后为${this._obj[k]}-${obj[k]}`)
        }
    }

    function defineActive(obj) {
        Object.keys(obj).forEach(k => {
            if (typeof obj[k] != "object") {
                _defineAttr(obj, k, obj[k])
                return;
            }
            defineActive(obj[k])
        })
        function _defineAttr(obj, key, value) {
            const dep = new Dep()
            Object.defineProperty(obj, key, {
                configurable: true,
                enumerable: true,
                get: () => {
                    dep.addSub(watcher1, obj, key, value)
                    dep.addSub(watcher2, obj, key, value)
                    return value
                },
                set: (_value) => {
                    if (_value === value) return;
                    value = _value
                    dep.notify(obj, key)
                }
            })
        }
    }
    const watcher1 = new Watcher("watcher1")
    const watcher2 = new Watcher("watcher2")
    const obj = {
        name: "cq",
        person: {
            habits: "music"
        }
    }
    defineActive(obj)
    obj.person.habits    //订阅
    obj.person.habits = "football"
    //输出
    //watcher1: 对象的habits属性改变了，更新后的前后为music-football
    //watcher2: 对象的habits属性改变了，更新后的前后为football-football
```
