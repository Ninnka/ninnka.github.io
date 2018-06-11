---
title: mint-ui的loadmore（上滑，下拉）组件的非侵入式（不修改源码）优化
date: 2018-03-17 22:57:58
categories:
	- 前端
	- JavaScript
tags:
	- Vue
	- 组件
	- mint-ui
	- 下拉刷新
	- 上拉加载
	- 优化
---

> 移动端开发时难免会有列表，列表最典型的的分页模式就是上滑加载更多，此外还需要刷新列表——下拉刷新数据。

最初，为了列表的上滑加载更多和下拉刷新数据的功能，我把目光放到了 `mint-ui` 的 `loadmore` 组件，虽说这个库已经有相当长的一段时间没有优化了。

移动端开发大部分时候还会遇到顶部一排tab，页面滑动可以切换tab这种需求，是的，每天中午点开的饿了么那一种，熟悉Android的同学大概知道那个叫做 `TabLayout` 和 `ViewPager`。但是这次的重点不是这个，而是我在使用 `loadmore` 的下拉加载时与我个人封装的 `swipe-tab-layout` 进行组合使用时体验不太理想，尤其是下拉太过容易触发，本想看看文档，查看组件是否有一个角度属性，用来设置触发下拉。不过很遗憾，并没有这种属性。

这种情况当然是只能查看源码，看看有没有能够修改的。果不其然，在查看 `loadmore` 组件的源码时发现了两个重要的方法：
1. handleTouchStart
2. handleTouchMove

故名思议，第一个方法是处理手指刚触摸到屏幕时的处理，第二个方法是处理手指在屏幕上滑动时的情况。

<!-- more -->

刚刚已经提到过了，现在需要为 `loadmore` 组件加入角度的判断功能，所以我尝试着修改了代码

```javascript
export default {
  handleTouchStart (event) {
    // 存储手指触摸的横坐标
    this.startX = event.touches[0].clientX;
    // ...
  },

  handleTouchMove (event) {
    // ...
    const tmpCurrentX = event.touches[0].clientX;
    const tmpCurrentY = event.touches[0].clientY;
    const diffX = tmpCurrentX - this.startX;
    const diffY = tmpCurrentY - this.startY;
    // 设置滑动的角度应该小与20°
    if (Math.abs(diffX) / Math.abs(diffY) <= Math.tan(20 * Math.PI / 180)) {
      event.stopPropagation();
      this.currentX = tmpCurrentX;
      this.currentY = tmpCurrentY;
      // ... 接下来是原本方法的处理逻辑
    }
  }
}
```

经过初步测试，这个方法确实完成了下拉时的角度限制。但是仍然有缺陷，那就是角度的判断是用的最新点与一开始的点进行的对比，如果一开始不满足滑动条件但是中途又满足了，这种情况仍然会触发下拉，所以我继续修改了代码：

```javascript
export default {
  methods: {
    handleTouchStart (event) {
      // 存储手指触摸的横坐标
      this.startX = event.touches[0].clientX;
      // ...
      this.startScrollCheck = true;
      this.startScrolling = false;
    },

    handleTouchMove (event) {
      if (!this.startScrollCheck) {
        return;
      }
      // ...
      const tmpCurrentX = event.touches[0].clientX;
      const tmpCurrentY = event.touches[0].clientY;
      const diffX = tmpCurrentX - this.startX;
      const diffY = tmpCurrentY - this.startY;
      // 设置滑动的角度应该小与20°
      if (Math.abs(diffX) / Math.abs(diffY) <= Math.tan(20 * Math.PI / 180) || this.startScrolling) {
        event.stopPropagation();
        this.startScrolling = true;
        this.currentX = tmpCurrentX;
        this.currentY = tmpCurrentY;
        // ... 接下来是原本方法的处理逻辑
      } else {
        !this.startScrolling && (this.startScrollCheck = false);
      }
    }
  },   
}
```

我对下拉做了进一步的处理，当滑动开始时先设置 `startScrollCheck` 标识滑动的开始，如果角度小与20°，则可以确认是进行下拉的动作，并设置 `startScrolling` 标识滑动进行中，这个时候只要有向下的动作，即使是170°的角度，只要有向下滑动的距离都会继续下滑。
这样一来便完成了下拉的角度限制优化。

但是很快我就面临了别的问题，得益于持续集成和持续部署，每当我提交代码的项目仓库时，脚本都会自动打包项目并部署到服务器上，每次打包都会重装 `node_modules`，换言之，刚刚的改 `node_modules` 源码形式只适用于本地开发。

硬着头皮把 mint-ui 源码 fork 了一份，在本地修改好，本打算测试成功后发布到私人团队用的npm，但是竟然没有办法build，node_modules 也没办法安装完全，看了一下package.json，猜测原因大概是有些package是element团队内部使用的吧。（继续寻求别的方法

随后，我把目光放在了runtime时的vue实例上。首先，我肯定能够获取到loadmore的实例，通过`ref` 属性，此外，我准备好上述修改完的方法，在组件 `mounted` 时，我便可以将修改的方法绑定到实例内部，以此来覆盖原有的方法。综上所述，我封装了一个新的 `list.vue` 组件，内部则是使用了 `loadmore`。

```html
<!-- list.vue template -->
<div class="list--component">
  <mt-loadmore ref="loadmore" :top-method="loadTop" :bottom-method="loadBottom" :bottom-all-loaded="allLoaded" :autoFill='autoFill' :topDistance="70" :bottomDistance="70">
    <slot name="list-items"></slot>
  </mt-loadmore>
</div>
```


```javascript
// list.vue <script>
export default {
  // ...
  async mounted () {
    function handleTouchStartCover (event) {
      this.startX = event.touches[0].clientX;
      this.startY = event.touches[0].clientY;
      this.startScrollTop = this.getScrollTop(this.scrollEventTarget);
      this.bottomReached = false;
      this.startScrollCheck = true;
      this.startScrolling = false;
      if (this.topStatus !== 'loading') {
        this.topStatus = 'pull';
        this.topDropped = false;
      }
      if (this.bottomStatus !== 'loading') {
        this.bottomStatus = 'pull';
        this.bottomDropped = false;
      }
    }

    function handleTouchMoveCover (event) {
      if (!this.startScrollCheck) {
        return;
      }
      if (this.startY < this.$el.getBoundingClientRect().top && this.startY > this.$el.getBoundingClientRect().bottom) {
        return;
      }
      const tmpCurrentX = event.touches[0].clientX;
      const tmpCurrentY = event.touches[0].clientY;
      const diffX = tmpCurrentX - this.startX;
      const diffY = tmpCurrentY - this.startY;
      if (Math.abs(diffX) / Math.abs(diffY) <= Math.tan(20 * Math.PI / 180) || this.startScrolling) {
        event.stopPropagation();
        this.startScrolling = true;
        this.currentX = tmpCurrentX;
        this.currentY = tmpCurrentY;
        var distance = (this.currentY - this.startY) / this.distanceIndex;
        this.direction = distance > 0 ? 'down' : 'up';
        if (typeof this.topMethod === 'function' && this.direction === 'down' &&
          this.getScrollTop(this.scrollEventTarget) === 0 && this.topStatus !== 'loading') {
          event.preventDefault();
          event.stopPropagation();
          if (this.maxDistance > 0) {
            this.translate = distance <= this.maxDistance ? distance - this.startScrollTop : this.translate;
          } else {
            this.translate = distance - this.startScrollTop;
          }
          if (this.translate < 0) {
            this.translate = 0;
          }
          this.topStatus = this.translate >= this.topDistance ? 'drop' : 'pull';
        }

        if (this.direction === 'up') {
          this.bottomReached = this.bottomReached || this.checkBottomReached();
        }
        if (typeof this.bottomMethod === 'function' && this.direction === 'up' &&
          this.bottomReached && this.bottomStatus !== 'loading' && !this.bottomAllLoaded) {
          event.preventDefault();
          event.stopPropagation();
          if (this.maxDistance > 0) {
            this.translate = Math.abs(distance) <= this.maxDistance
              ? this.getScrollTop(this.scrollEventTarget) - this.startScrollTop + distance : this.translate;
          } else {
            this.translate = this.getScrollTop(this.scrollEventTarget) - this.startScrollTop + distance;
          }
          if (this.translate > 0) {
            this.translate = 0;
          }
          this.bottomStatus = -this.translate >= this.bottomDistance ? 'drop' : 'pull';
        }
        this.$emit('translate-change', this.translate);
      } else {
        !this.startScrolling && (this.startScrollCheck = false);
      }
    }

    this.$refs.loadmore.$el.removeEventListener('touchstart', this.$refs.loadmore.handleTouchStart);
    this.$refs.loadmore.$el.removeEventListener('touchmove', this.$refs.loadmore.handleTouchMove);

    // 通过bind方法将实例原有的方法覆盖
    this.$refs.loadmore.handleTouchStart = handleTouchStartCover.bind(this.$refs.loadmore);
    this.$refs.loadmore.handleTouchMove = handleTouchMoveCover.bind(this.$refs.loadmore);

    this.$refs.loadmore.$el.addEventListener('touchstart', this.$refs.loadmore.handleTouchStart);
    this.$refs.loadmore.$el.addEventListener('touchmove', this.$refs.loadmore.handleTouchMove);
  },
  // ...
}
```

经过优化后，与 `swipe-tab-layout` 的组合使用有了不错的体验。



























