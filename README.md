### vuex学习笔记
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
