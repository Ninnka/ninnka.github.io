---
title: vue-pwd-input 支付密码输入组件的实现
date: 2018-04-16 13:08:02
categories:
	- 前端
	- JavaScript
tags:
	- Vue
	- mobile-component
	- password-input
---

> 这次记录的是一个类似移动端支付密码输入框组件的实现。当然，这种组件并不局限于在移动端使用，在PC端也可以使用，比如：在京东，淘宝下单输入支付密码时。
> [仓库地址](https://github.com/Rennzh/vue-pwd-inputs)

## 初步设计

既然要用来输入密码，那肯定是需要输入框的，假设默认输入位数是6位，那么需要准备6个输入框。

```html
<!-- pwd-inputs.vue -->
<div class="inputs">
  <div class="input-wrap" v-for="(pwd, index) of pwdLength" :key="index">
    <input type="password" maxlength="1" v-model="pwds[index]">
  </div>
 </div>
```

<!-- more -->

```javascript
// pwd-inputs.vue
export default {
  name: 'PwdInputs',
  
  props: {
  },
  
  data () {
    return {
      pwds: {
        type: Array,
        default () {
          return ['', '', '', '', '', ''];
        }
      },
    }
  }
}
```

```less
  // pwd-inputs.vue
  .inputs {
    height: 50px;
    display: flex;
    align-items: center;
    justify-content: space-around;

    > .input-wrap {
      height: 40px;
      width: 40px;
      font-size: 16px;
      text-align: center;
      flex-grow: 0;
      flex-shrink: 1;
      border: 1px solid #be9f79;
      border-radius: 6px;
      box-sizing: border-box;
      display: flex;
      align-items: center;

      input {
        outline: none;
        border: none;
        height: 20px;
        font-size: 16px;
        width: 100%;
        text-align: center;
      }
    }
  }
```

初步完成后，我测试了一下组件，发现了有了个问题，（先不讨论这个组件如何与父组件进行数据交流）在移动端中，password的输入框都是有1-2秒的显示时间，也就是输入密码后，密码会有1-2秒的时间显示了你输入了什么。从输入体验上而言，如果是登录的密码输入这或许很不错，但是对于只有6位或者更短的密码而言，这就显得无太多意义了。所以下面将改造组件，并增加父子组件数据交流的方法。

## 查漏补缺

经过初步完成后的测试，我发现了以下几个问题

1. 没有父子组件的数据交流。
2. password输入框的输入体验问题。

那么针对以上这些问题，对组件进行升级。

```html
<!-- pwd-inputs.vue -->
<div class="inputs">
  <div class="input-wrap" v-for="(pwd, index) of pwdLength" :key="index">
    <input type="text" maxlength="1" :ref="refNames[index]" :data-input-index="index + 1" :value="pwds[index]" @input="afterInput">
  </div>
 </div>
```

首先把模板中的input type改为 `text`，增加一个 `data` 属性 `input-index`

```javascript
// pwd-inputs.vue
export default {
  name: 'PwdInputs',
  
  props: {
    pwdLength: {
      type: Number,
      default: 6
    },
    pwds: {
      type: Array,
      default () {
        return ['', '', '', '', '', ''];
      }
    },
    pwdTexts: {
      type: Array,
      default () {
        return ['', '', '', '', '', ''];
      }
    },
    currentFocusIndex: {
      type: Number,
      default: 0
    },
  },
  
  data () {
    return {
    }
  }
}
```

既然password的输入框会有显示延迟，那么可以直接换成text的输入框。
为了方便控制input，增加一个当前聚焦的input的index，
并且密码数组由父组件通过prop传入组件中。

此外，还需要增加组件内的输入框输入事件响应方法，当有字符输入时，将输入的字符通过事自定义件传递到父组件。

```html
<!-- pwd-inputs.vue -->
<div class="inputs">
  <div class="input-wrap" v-for="(pwd, index) of pwdLength" :key="index">
    <input type="text" maxlength="1" :data-input-index="index + 1" :value="pwds[index]">
  </div>
 </div>
```


```javascript
// pwd-inputs.vue
export default {
  name: 'PwdInputs',
  
  props: {
    pwdLength: {
      type: Number,
      default: 6
    },
    refNames: {
      type: Array,
      default () {
        return [
          'input1',
          'input2',
          'input3',
          'input4',
          'input5',
          'input6',
        ];
      }
    },
    pwds: {
      type: Array,
      default () {
        return ['', '', '', '', '', ''];
      }
    },
    pwdTexts: {
      type: Array,
      default () {
        return ['', '', '', '', '', ''];
      }
    },
    currentFocusIndex: {
      type: Number,
      default: 0
    },
  },
  
  data () {
    return {
    }
  }，
  
  methods: {
    afterInput (event) {
      // console.log('afterInput event', event);
      if (event.target.dataset && event.target.value !== undefined && event.target.value !== null && event.target.value.trim() !== '') {
        const index = Number(event.target.dataset.inputIndex);
        this.$emit('pwdTextsChange', { index: index - 1, value: event.target.value });
        this.$emit('pwdsChange', { index: index - 1, value: '*' });
        event.target.value = '*';
        if (index === this.pwdLength) {
          return;
        }
        this.$nextTick(() => {
          this.$refs[this.refNames[index]][0].focus();
          this.$emit('currentFocusIndexChange', index);
        });
      }
    },
  }
}
```

为了方便获取input的DOM元素，增加了一个引用名的数组，可以由外部组件传入，也可以使用默认的引用名。在input触发输入事件时，将输入后的变化通过自定义事件传递到父组件。

```html
<!-- parent-comp.vue -->
<div>
  <inputs
    class="inputs-style"
    ref="inputs"
    :pwdTexts="pwdTexts"
    :pwds="pwds"
    :currentFocusIndex="currentFocusIndex"
    @pwdTextsChange="pwdTextsChange"
    @pwdsChange="pwdsChange"
    @currentFocusIndexChange="currentFocusIndexChange">
  </inputs>
</div>
```

```javascript
// parent-comp.vue
import Inputs from '@/components/pwd-inputs';

export default {
  // ...
  components: {
    // ...
    Inputs
  },
  
  data () {
    return {
      refNames: [
        'input1',
        'input2',
        'input3',
        'input4',
        'input5',
        'input6',
      ],
      pwdTexts: ['', '', '', '', '', ''],
      pwds: ['', '', '', '', '', ''],
      currentFocusIndex: 0,
    }
  },
  
  methods: {
    // ...
    pwdTextsChange ({ index, value }) {
      this.pwdTexts[index] = value;
    },

    pwdsChange ({ index, value }) {
      this.pwds.splice(index, 1, value);
    },

    currentFocusIndexChange (index) {
      this.currentFocusIndex = index;
    },
  }
  // ...
}
```

除了上述的问题外，可能还缺少焦点控制的问题，所以还需要提供焦点控制的方法。

```javascript
// pwd-inputs.vue
export default {
  // ...
  methods: {
    focusInStart () {
      // 聚焦到第一个input
      try {
        this.$refs[this.refNames[0]][0].focus();
      } catch (err) {
        throw err;
      }
    },

    focusInEnd () {
      // 聚焦到最后一个input
      try {
        this.$refs[this.refNames[this.pwdLength - 1]][0].focus();
      } catch (err) {
        throw err;
      }
    },

    focusTargetInput (target) {
      // 聚焦到指定的input
      try {
        this.$refs[this.refNames[target]][0].focus();
      } catch (err) {
        throw err;
      }
    },
  },
  // ...
}
```

完善之后组件基本成型了，当然，还有一些别的功能暂时没有想到或者还不需要。
等有必要的时候会继续增加新功能。
