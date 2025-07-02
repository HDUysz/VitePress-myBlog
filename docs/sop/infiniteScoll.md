---
top: 3
---

# 📚 React虚拟列表踩坑记录

> 记录一下在项目中使用各种虚拟列表库时踩的坑，涉及三个流行库的真实使用体验。

## 📊 库对比总览

| 库名称 | 周下载量 | 维护状态 | 双向滚动支持 | 主要问题 |
|--------|----------|----------|--------------|----------|
| [Virtuoso](https://virtuoso.dev/) | - | ✅ 活跃维护 | ⚠️ `startReached` 有问题 | 向上滚动无法正常触发 |
| [react-infinite-scroller](https://github.com/danbovey/react-infinite-scroller) | 39万+ | ❌ 4年未维护 | ✅ 支持 `isReverse` | 坑爹中的坑爹，会无限循环 |
| [react-infinite-scroll-component](https://github.com/ankeetmaini/react-infinite-scroll-component) | 90万+ | ❌ 4年未维护 | ✅ 完美支持 | 内容不足时无法触发加载 |

---

## 🎯 Virtuoso

项目里面原本集成的是[Virtuoso](https://virtuoso.dev/)，这个虚拟列表确实足够好用，正常使用的时候如下所示：

```jsx
<Virtuoso
  data={dataSource}
  endReached={() => {
    fetchNextPage();
  }}
  itemContent={(index, item) => (
    <Flexbox key={item.id} paddingInline={12}>
      <ChunkItem {...item} index={index} />
    </Flexbox>
  )}
/>
```

只需要设置他的`endReached`，用来请求下一页数据就可以了。然而，这个项目有一个很大的问题，就是虽然他的文档里面写着支持向上翻页，也就是`startReached`。原文是这样说的：

> The components expose startReached and endReached callback properties, suitable for loading data on demand. 

然而，当我们将`endReached`替换为`startReached`时，却发现无论页面如何向上滑动，都无法触发向上请求，而且发现，当刷新页面的时候，莫名会触发到`startReached`的钩子，由于作者没有开源此项目，看不到源码，因此也就不得而知这个作者是如何进行判定的了。

这时候，看到github中有人提出了这样一个奇思妙想，将整个页面反转过来，利用：

```css
flex-direction: column-reverse;
```

就可以实现用户向上翻动页面的时候，触发的是`endReached`的钩子，详情可以去这个项目的`issue`里面进行查看，非常有趣。但是这个实现方法，始终不是我希望实现的方式。

---

## ⚠️ react-infinite-scroller

我第二个尝试使用的，就是这个组件，作为一个周下载量超过39万的库，居然已经将近四年没有人进行维护，`issue`里面也堆满了各种bug，使用方式大致如下:

```jsx
<div style="height:700px;overflow:auto;">
    <InfiniteScroll
        pageStart={0}
        loadMore={loadFunc}
        hasMore={true || false}
        loader={<div className="loader" key={0}>Loading ...</div>}
        useWindow={false}
        isReverse={true}
    >
        {items}
    </InfiniteScroll>
</div>
```

其中`isReverse`可以控制向上翻动进行请求。最坑的地方来了，这个组件简直是坑爹中的坑爹，当向上请求翻页的时候，会将scollTop置零，导致页面一直会被强行拉倒最上方，导致又会触发向上请求页面，从而循环下去。。

在github中，有人提出可以使用：

```css
overflow-anchor: none;
```

来进行解决，然而当我尝试的时候发现，似乎这个解决方法只能解决向下翻动，而无法解决向上翻动的问题。我手动监控页面，使用js控制滚动条，发现无法阻止翻滚到顶部的默认行为，会出现回到顶部又跳下来的情况，遂不使用该组件。

---

## ✅ react-infinite-scroll-component

这是一个周下载量超过90万的热门虚拟列表库，它完美解决了我向上翻动的问题，然而这同样是一个年久失修的库，已经超过至少4年没有维护。于是它存在一个需要我们手动维护的bug:

当我们初次加载的message，无法填充满整个容器，就会导致向上请求失效，向下请求同样的道理。

向上翻动，有人提出了一个很好的[参考方法](https://github.com/ankeetmaini/react-infinite-scroll-component/issues/399)

我这里补充一个向上翻动的时候的函数：

```ts
import { useDebounceFn } from 'ahooks';
import { useEffect } from 'react';

/* eslint-disable @typescript-eslint/member-ordering */

interface UseFixScrollOptions {
  containerId?: string;
  debounceWait?: number;
  fetchMore: () => void;
  hasMore: boolean;
}

/**
 * 修复 react-infinite-scroll-component 的已知问题：
 * "Don't load new items when content height <= screen height"
 *
 * 参考：https://github.com/ankeetmaini/react-infinite-scroll-component/issues/391
 *
 * 原理：
 * 1. 使用 MutationObserver 监听消息容器的DOM变化
 * 2. 当检测到新消息添加后，检查loader是否在可视区域内
 * 3. 如果loader可见且用户未滚动，则强制触发fetchMore
 * 4. 持续这个过程直到内容填满屏幕或没有更多消息
 */
export function useFixScroll({
  hasMore,
  fetchMore,
  containerId = 'scrollableDiv',
  debounceWait = 500,
}: UseFixScrollOptions) {
  // 防抖的检查函数，避免频繁触发
  const fixFetch = useDebounceFn(
    () => {
      // eslint-disable-next-line unicorn/prefer-query-selector
      const scrollContainer = document.getElementById(containerId);
      if (!scrollContainer) {
        return;
      }

      // 查找loader元素（loading indicator）
      const loaderElement = scrollContainer.querySelector('.infinite-scroll-component-loader');
      if (!loaderElement) {
        return;
      }

      const scrollContainerRect = scrollContainer.getBoundingClientRect();
      const loaderRect = loaderElement.getBoundingClientRect();

      // 核心检查逻辑：
      // 1. 用户没有主动滚动（scrollTop === 0）
      // 2. loader元素在可视区域内（loader的top < 容器的bottom）
      const shouldFetch =
        scrollContainer.scrollTop === 0 && loaderRect.top < scrollContainerRect.bottom;

      if (shouldFetch && hasMore) {
        fetchMore();
      }
    },
    {
      wait: debounceWait,
    },
  );

  useEffect(() => {
    if (!hasMore) {
      return;
    }

    // eslint-disable-next-line unicorn/prefer-query-selector
    const scrollContainer = document.getElementById(containerId);
    if (!scrollContainer) {
      return;
    }

    // 创建 MutationObserver 监听DOM变化
    const observer = new MutationObserver((mutationsList) => {
      for (let mutation of mutationsList) {
        if (mutation.type === 'childList') {
          fixFetch.run();
        }
      }
    });

    // 配置观察选项：只监听直接子节点变化
    const config = {
      childList: true, // 监听子节点的添加/删除
      subtree: false, // 不监听整个子树，避免过度触发
    };

    observer.observe(scrollContainer, config);

    // 组件挂载后立即检查一次
    fixFetch.run();

    return () => {
      observer.disconnect();
      fixFetch.cancel();
    };
  }, [hasMore, fetchMore, containerId, fixFetch]);
}
```

使用方法如下:

```ts
// 修复 react-infinite-scroll-component 在内容高度小于屏幕高度时不自动加载的问题
useFixScroll({
  containerId: 'scrollableDiv',
  fetchMore: handleLoadMore,
  hasMore: messageHasMore && !isMessageLoadingMore && messageEndId !== '0',
});
```

至此，至少是加载历史消息的功能，算是能用了。大概是看了至少一天才解决了上面的问题。。最后还是借用的第三方方法来解决，然而这些第三方也存在这里那里的问题，解决起来还是要一点功夫的。
