---
title: Vue2的组件继承
date: 2017-06-05 11:41:31
categories:
	- Web
	- JavaScript
tags:
	- Vue
	- extend
---

> 接触过面向对象语言的同学肯定清楚，面向对象中有一个非常重要的特性，那就是继承。通过继承可以更好的封装一个类，良好的封装能够减少耦合，并且可以对成员进行更精确的控制，Vue也提供了类型的继承方法。

## 一个基础的状态组件

在考虑继承组件前，我们先创建一个基础的状态组件。组件的运用场景比较简单，有一个列表，列表的每一项有自己的状态，这个状态组件显示在列表中每一项的右上角或者左上角，并显示出具体的状态文本。

```html
<!-- status.vue -->
<div class="item-status" :style="{
    ...statusStyles,
    ...positionStyles,
}">
    <span>{{ statusText }}</span>
</div>
```

<!-- more -->

```javascript
// status.vue
export default {
    name: 'ItemStatus',
    
    props: {
        status: {
            type: [Number, String],
            default: '',
        },
        
        statusText: {
          type: String,
          default: '',
        },
        
        visible: {
            type: Boolean,
            default: true
        },
    },
    
    data: {
        return {

        };
    },
    
    computed: {
        statusStyles () {
            return {};
        },
        
        positionStyles () {
            return {};
        },
    },
}
```

```less
// status.vue 粗略的样式大概是这样
.item-status {
	position: absolute;
    width: 30px;
    height: 30px;
    background-color: #979797;
    color: #ffffff;
    text-align: center;
    // ...
}
```

创建好组件后，我们可以在任意位置引入并使用，比如：

```html
<!-- item.vue -->
<div class="item">
    <status :statusText="statusTextGetter"></status>
</div>
```

```javascript
// item.vue
export default {
    name: 'ListItem',
    
    props: {
        itemData: {
            type: Object,
            default () {
                return {};
            }
        },
    },
    
    data: {
        return {};
    },
    
    computed: {
        statusTextGetter () {
            return this.itemData.status ? this.itemData.status : '';
        },
    },
}
```

```less
// item.vue
.item {
    position: relative;
    // ...
}
```

以上就完成了就基础的组件组合，但是现在有一个新的需求，就是不同页面有不同的多个列表，列表有各自的状态，不同的状态显示不同的背景颜色，不同的列表的状态形状不同，此外还有一些别的特性。你可能想到把他们分开来，独立成两个不同组件，又或者把所有状态通过props塞到一个组件中逐个判断，其实两种方法都不是很妥当，我认为这里可以使用Vue的extend属性对单文件组件进行扩展是比较妥当的做法。

## 使用extend

### extend需要注意以下两个点：

- template只能完全覆盖或者完全继承
- 如果定义了相同的方法和属性，则新的会覆盖原组件定义的方法和属性
- 生命周期方法类似余mixin，会合并，原组件和继承之后的组件都会被调用，原组件先调用

在注意以上两点后，我们开始创建两个ListItem组件，分别用在两个不同的列表

```javascript
// status1.vue 状态位置在左上角，status:[1, 2, 3, 4]
import status from '@/components/status';
export default {
    name: 'status1',
    
    extend: status,
    
    prop: {
        topOffset: '0',
        leftOffset: '0',
    },

    data: {
        return {
            statusList: [1, 2, 3, 4]
        };
    },
    
    computed: {
        statusStyles () {
        	const styles = {};
        	switch (this.status) {
                case 1:
                    // ...
                    break;
                // ...
                default:
        	}
            return styles;
        },
        
        positionStyles () {
            return {
                top: this.topOffset,
                left: this.leftOffset,
            };
        },
    },
}
```

```javascript
// status2.vue 状态位置在左上角，status:[5, 6, 7, 8]
import status from '@/components/status';
export default {
    name: 'status2',
    
    extend: status,
    
    prop: {
        topOffset: '0',
        rightOffset: '0',
    },

    data: {
        return {
            statusList: [5, 6, 7, 8]
        };
    },
    
    computed: {
        statusStyles () {
        	const styles = {};
        	switch (this.status) {
                case 5:
                    // ...
                    break;
                // ...
                default:
        	}
            return styles;
        },
        
        positionStyles () {
            return {
                top: this.topOffset,
                right: this.rightOffset,
            };
        },
    },
}
```

创建好组件后，使用方法与原先相同。

## 结论

- 如果的形态组件类似，并且使用广泛时，可以优先考虑创建一个基础组件，再用extend扩展出几个更加具体的组件，基础组件的模板需要有较强的可配置性，不然无法适应不同的形态变化。
- 除了使用extend外，使用props传入参数，在组件内根据参数判断使用哪一种组件形态也是可以的，前提是组件内不能太过复杂，如果组件内的复杂度太高，根据运用场景的不同，形态也多变，那么这种方法会把大量工作压在一个组件内，不利于扩展与维护，子组件与父组件的耦合也会非常严重，我们更加需要能灵活运用的组件。