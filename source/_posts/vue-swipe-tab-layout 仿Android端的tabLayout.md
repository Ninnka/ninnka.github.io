---
title: vue-swipe-tab-layout 仿Android端的 TabLayout + ViewPager
date: 2017-05-03 19:08:02
categories:
	- Web
	- JavaScript
tags:
	- Vue
	- mobile-component
	- swipe-tab-layout
---

> 公司最近要开发新的移动端项目，项目中有很多页面都用到了tab分组。
> 想起在 Android 中看到的 TabLayout + ViewPager 的组合，决定动手做一个类似的 Vue 组件。
> [github地址](https://github.com/Rennzh/vue-swipe-tab-layout)

## 组件设计

> 组件的功能基本参照 Android 端，首先把组件的设计要点列出来

- 顶部是一个可以横向的 tab-nav 容器；容器内需要填充 nav-item；使用者可以自定义 nav-Item 的具体内容，然后通过 template 与 slot 分发到这个容器中。
- tab-nav 容器有两种模式：
  1. nav-item 会自动扩展，平分容器宽度(每一个 nav-item 的宽度是一样的)。
  2. nav-item 不会自动扩展，每一个 nav-item 的宽度由使用者决定。
- tab-nav 容器内有导航条，nav-indicator (导航条) 可以在 tab-nav 会根据当前 nav-item 的 激活状态变化来滑动；可以定义 nav-indicator 的宽度缩放比，缩放比相对所在的 nav-item 的宽度；可以定义 nav-indicator 的位置(顶部或者底部)；可以设置 nav-indicator 的位置偏移值；
- 除去顶部的 tab-nav，剩下的就是内容 tab-content，tab-content 的容器水平排列，宽度为浏览器可视区域的宽度。
- tab-content 的具体DOM结构同样通过 template 与 slot 分发到组件中。
- tab-content的父级可以水平滑动，滑动使用 translate3d。
  - 在第一屏手指向右滑动时，禁止滑动，手指向左时可滑动，先向右再向左可以触发滑动。
  - 在最后一屏手指向左滑动时，禁止滑动，手指向右滑动时可滑动，先向左再向右可以触发滑动。
  - 除去上述两种情况，其他均可任意滑动。
  - 触发滑动到下一页(前一页)的条件默认是手指滑动距离为屏幕宽度1/3以上。
- tab-content 滑动时，nav-indicator 需要跟着一起滑动
- nav-indicator 滑动到目标位置后，如果当前 nav-item 不在浏览器可视区域的正中间，tab-nav 容器会尽可能的滑动使当前 nav-item 到达可视区域的正中间，如果允许的滑动距离不够的话，使用最大的可滑动距离。

在实际写代码前，尽可能的列出需求可以防止开发中突然发现新需求而不得不破坏当前结构的情况。(说的就是我o(╥﹏╥)o)
下面开始分析代码如何写。

<!-- more -->

## 组件代码分析

- 首先分析模板及内容的 slot

```html
<!-- swipe-tab-container.vue -->
<div :data-owner="owner" class="swipe-tab-container">
    <div class="swipe-tab-nav--layer" ref="swipeTabNavLayerRef">
      <div class="swipe-tab-nav--slider" ref="swipeTabNavSliderRef" :style="swipeTabNavSliderStyleGetter">
        <div
          class="nav-item--wrapper"
          :class="fullFlex ? 'flex-wrapper' : ''"
          v-for="(tabNav, index) of tabNavList"
          :key="tabNav.key"
          ref="tabNavRef">
          <slot :name="`swipe-tab-nav-${tabNav.key}`"></slot>
        </div>
        <div class="nav-indicator" :style="indicatorStyleGetter"></div>
      </div>
    </div>
    <div class="swipe-tab-content--layer">
      <div class="swipe-tab-content--slider" :style="sliderStyleGetter">
        <div class="swipe-tab-content--item" v-for="tabNav in tabNavList" :key="tabNav.key">
          <slot :name="`swipe-tab-content-${tabNav.key}`" class="tab-content--slot"></slot>
        </div>
      </div>
    </div>
</div>
```

```javascript
// swipe-tab-container.vue
export default {
    name: 'SwipeTabContainer',
    
    props: {
        // * 当前组件的拥有者，一般设置为所在的路由页面或者父组件
        owner: '',

        // * tab对象列表
        /**
         [
             {
                label: 'label',
                key: 'key',
                type: 0, optional
             }
         ] 
         **/
        tabNavList: {
          type: Array,
          default () {
            return [];
          }
        },
        
        fullFlex: false, // 控制 nav-item 的样式，false是第二种模式，true是第一种模式
    },

    data () {
        return {}
    },
}
```

```less
    // 具体样式可以查看代码

    .swipe-tab-content--layer {
      // ...

      .swipe-tab-content--slider {
        // ...

        * {
          // 如果有需要水平的滚动元素必须另外设置touch-action: auto;
          // 设置 touch-action: pan-y; 是为了解决safari上的滚动问题。
          touch-action: pan-y;
        }
      }
    }
```

无论是 nav-item 还是 tab-content 都使用 slot 分发，能够最大限度的允许使用者自定义内容。其他则属于组件自身的行为。

现在已经可以分发 DOM 节点了，但是 tab-nav 容器的宽度在第二种模式是有问题的。
内部的div的宽度如果自然伸展，最大不会比它的容器大，所以，在第二种模式下是需要计算子元素的宽度并最终得出容器的最小宽度应该是多少，这里面会有一点点的偏差。
但是保险起见，在两种模式下都进行计算，并判断那个比较数值更大，然后使用数值大的那一个。
此外，因为 nav-indicator 和 tab-content 容器滑动时是要计算滑动的距离的，为了避免在滑动时的大量计算导致滑动掉帧，在初始化前将需要的值计算好。
初始化时还需要添加好手指滑动的事件，这里我使用了 `hammerjs` 作为手势库。

```javascript
// swipe-tab-container.vue
export default {
    name: 'SwipeTabContainer',
    
    // ...
    
    async mounted () {
        const swipeContentLayer = document.querySelector(`[data-owner=${this.owner}] .swipe-tab-content--layer`);
        const swipeTabContentSlider = document.querySelector(`[data-owner=${this.owner}] .swipe-tab-content--slider`);
        this.swipeContentLayerWidth = swipeContentLayer.getBoundingClientRect().width;
        if (swipeContentLayer && this.$refs.swipeTabNavSliderRef) {
          this.swipeContentTabNavSliderWidth = this.$refs.swipeTabNavSliderRef.getBoundingClientRect().width;
          // * 设置slider容器的宽度
          const tmpWidth = this.tabNavList.length * swipeContentLayer.getBoundingClientRect().width;
          this.swipeContentSliderWidth = !isNaN(tmpWidth) ? tmpWidth : null;
          // * 设置slider容器的滑动监听
          this.sliderHammerIns = new Hammer(swipeTabContentSlider);
          this.sliderHammerIns.get('tap').set({ enable: false });
          this.sliderHammerIns.get('press').set({ time: 50 });
          this.sliderHammerIns.on('press', this.sliderPress);
          this.sliderHammerIns.get('pan').set({ direction: Hammer.DIRECTION_HORIZONTAL });
          this.sliderHammerIns.on('panstart', this.sliderPanStart, true);
          this.sliderHammerIns.on('panend', this.sliderPanEnd);
          this.sliderHammerIns.on('panleft', this.sliderPanLeft, true);
          this.sliderHammerIns.on('panright', this.sliderPanRight, true);
          this.sliderHammerIns.on('pancancel', this.sliderPancancel);

          setTimeout(() => {
            const tabNavRefs = this.$refs.tabNavRef;
            let tmpWidth = 0;
            let tmpLefts = [];
            this.navItemWidths = tabNavRefs.map((key, index) => {
              let tmpLeft = tabNavRefs[index].offsetLeft;
              const clientRect = tabNavRefs[index].getBoundingClientRect();
              const offsetWidth = clientRect.width;
              tmpWidth += offsetWidth;
              // * 对比left
              tmpLeft = ~~clientRect.left === tmpLeft ? clientRect.left : tmpLeft;
              tmpLefts.push(tmpLeft);
              return offsetWidth;
            });
            this.navItemLefts = [...tmpLefts];
            this.swipeContentTabNavSliderWidth = tmpWidth;
            this.maxtabNavSliderScrollXDiff = Math.abs(this.swipeContentTabNavSliderWidth - this.swipeContentLayerWidth);
            this.navItemWidths.forEach((item, index) => {
              if (index < tabNavRefs.length - 1) {
                const tmp1 = tabNavRefs[index].getBoundingClientRect().left;
                const tmp2 = tabNavRefs[index + 1].getBoundingClientRect().left;
                this.translateScales.push(Math.abs((tmp2 - tmp1) / this.swipeContentTabNavSliderWidth));
              }
            });
            this.navIndicatorTranslateX = this.navItemWidths[this.currentTabIndex] * this.indicatorWidthScaleFactor;
            this.navIndicatorTranslateXOld = this.navIndicatorTranslateX;
            this.initScroll();
          }, 1);
        }
    },

    data () {
        return {
          tabNavScrollIns: null,
          tabNavScrollOptions: {},
          swipeContentLayerWidth: null,
          swipeContentTabNavSliderWidth: null,
          swipeContentSliderWidth: null,
          swipeContentSliderTranslateXOld: 0,
          swipeContentSliderTranslateX: 0,
          panAllow: false,
          sliderHammerIns: null,
          sliderCssTransition: false,
          slidertransitionTime: 0.4,
          isPaningLeft: false,
          isPanedLeft: false,
          isPaningRight: false,
          isPanedRight: false,
          tmpLeftCenter: null,
          tmpRightCenter: null,
          setLeftDirection: false,
          setRightDirection: false,
          navItemWidths: [],
          navItemLefts: [],
          navIndicatorTranslateX: 0,
          navIndicatorTranslateXOld: 0,
          translateScales: [],
          maxtabNavSliderScrollXDiff: 0,
        }
    },
}
```

在初始化滑动实例时，我只添加了 水平滑动的监听，垂直滑动交由浏览器自身处理。
并且监听了滑动取消的事件，因为如果在滑动期间发生了意外导致滑动被迫中断时可以迅速根据当前的条件判断页面应该滑动到那一页。

```javascript
export default {
    // ...
    methods: {
        // 单击 nav-item
        tabNavClick (params) {
          this.$emit('tabNavClick', params);
          if (params.index !== this.currentTabIndex) {
            this.navChange(params);
          }
        },

        // 双击 nav-item
        tabNavdblClick (params) {
          this.$emit('tabNavdblClick', params);
        },

        // nav-item 变化
        navChange ({ index, tabNav }) {
          this.sliderCssTransition = true;
          this.$emit('update:currentTabIndex', index);
          this.navIndicatorSlide({ index, tabNav });
          this.navlayerSliderSlide({ index, tabNav });
          this.contentSlide({ index, tabNav });
        },

        // nav-indicator 滑动
        navIndicatorSlide ({ index, tabNav }) {
          const tabNavItemRef = this.$refs.tabNavRef[index];
          const clientRect = tabNavItemRef.getBoundingClientRect();
          let left = tabNavItemRef.offsetLeft;
          left = ~~clientRect === left ? ~~clientRect : left;
          this.navIndicatorTranslateX = left + clientRect.width * this.indicatorWidthScaleFactor;
          this.navIndicatorTranslateXOld = this.navIndicatorTranslateX;
        },

        // tab-nav 容器滑动
        navlayerSliderSlide ({ index }) {
          if (this.swipeContentTabNavSliderWidth <= this.swipeContentLayerWidth + 1.5) {
            // * 如果layerSlider的宽度小于等于layer加上偏差值，则不需要滚动slider
            return;
          }
          const width = this.navItemWidths[index];
          const left = this.navItemLefts[index];
          let diff = (left + width / 2) - this.swipeContentLayerWidth / 2;
          diff = diff > this.maxtabNavSliderScrollXDiff ? this.maxtabNavSliderScrollXDiff : diff;
          diff = diff < 0 ? 0 : -diff;
          this.tabNavScrollIns.scrollTo(diff, 0, 300);
        },

        // 滑动tab-content
        contentSlide ({ index, tabNav }) {
          this.swipeContentSliderTranslateX = -this.swipeContentLayerWidth * index;
          this.swipeContentSliderTranslateXOld = this.swipeContentSliderTranslateX;
        },

        // 更新tab-index
        updateCurrentTabIndex (index) {
          index = index > this.tabNavList.length - 1 ? this.tabNavList.length - 1 : index;
          index = index < 0 ? 0 : index;
          this.$emit('update:currentTabIndex', index);
        },

        // 调整预测的位置
        correctPredictIndex (index) {
          index = index > this.tabNavList.length - 1 ? this.tabNavList.length - 1 : index;
          index = index < 0 ? 0 : index;
          return index;
        },

        // * --------
        // 手指按压
        sliderPress (ev) {
          // console.log('sliderPress ev', ev);
          ev.preventDefault();
          this.sliderCssTransition = false;
        },
        
        // 手指开始滑动
        sliderPanStart (ev) {
          // console.log('sliderPanStart ev', ev);
          ev.preventDefault();
          ev.srcEvent.stopPropagation();
          const { angle } = ev;
          this.sliderCssTransition = false;
          if (Math.abs(angle) >= 145 || Math.abs(angle) <= 35) {
            // * 滑动角度大于160才算入有效滑动
            this.swipeContentSliderTranslateXOld = this.swipeContentSliderTranslateX;
            this.navIndicatorTranslateXOld = this.navIndicatorTranslateX;
            this.panAllow = true;
          } else {
            // console.log('stop');
            this.sliderHammerIns.stop();
          }
          return false;
        },

        sliderPanEnd (ev) {
          // ... 具体代码可以查看仓库
        },

        sliderPanRight (ev) {
          // ... 具体代码可以查看仓库
        },
        sliderPanRight (ev) {
          // ... 具体代码可以查看仓库
        },

        sliderPancancel (ev) {
          // console.log('sliderPancancel', ev);
          this.sliderPanEnd(ev);
        },

        scrollReset () {
          // * 初始化navIndicator的滚动
          this.navIndicatorTranslateX =
            this.navIndicatorTranslateXOld =
              this.navItemWidths[0] * this.indicatorWidthScaleFactor;
          // * 初始化tabContainer的滚动
          this.swipeContentSliderTranslateX = this.swipeContentSliderTranslateXOld = 0;
        }
    },
    // ...
}
```

swipe-tab-container 的重要代码就差不多完成了。
还剩下 swipe-tab-nav，就是 nav-item 的组件。
swipe-tab-nav 默认显示 对象的 label 属性，当然也可以通过 slot 把自定义的 DOM 传入到组件中。

```html
<!-- swipe-tab-nav.vue -->
<div class="swipe-tab-nav-item" ref="tabNavItem">
    <span v-if="$slots.default === undefined">{{ tabLabel }}</span>
    <slot v-else></slot>
</div>
```

```javascript
// swipe-tab-nav.vue
export default {
  name: 'SwipeTabNav',

  props: {
    tabLabel: ''
  },
  
  // ...
}
```

```less
.swipe-tab-nav-item {
  cursor: pointer;
}
```

## 遇到的一些问题

1. 不同浏览器的原生滚动和hammer的兼容问题以及 touch-action 属性

前面在样式中提到过，在safari中或者Android的微信与hammerjs有着奇奇怪怪的行为，滑动经常无故被中断，要不就是浏览器的原生滑动被中断。
解决办法就是如果内部有列表使用了浏览器的原生滚动overflow: auto，那么该元素需要 touch-action: pan-y | pan ，并且hammer实例不能监听 垂直方向上的pan事件。

2. DOM节点的width，height 以及 left，top 的精度问题

这其实是一个比较老生常谈的问题，DOM属性上的 width，height， left 和 top 一定是一个整数，所以使用这个属性时会缺失精度。建议使用 getBoundingClientRect。




























