# Docsify集成Gitalk

鉴于网上对于 Docsify 集成 Gitalk 的教程过于旧，而且大多教程存在一个问题，就是一个博客站点只对应一个 GitHub 中的 issue，但是一般的使用场景却是一个博客站点有多篇博客，需要的是每一篇文章对应 GitHub 上的一个 issue，网上对于这样的配置的讲解少之又少，现在让我来拨乱反正！

首先我们需要知道 Gitalk 的配置选项如下：

- `clientID`：申请的 OAuth App 的 Client ID
- `clientSecret`：申请的 OAuth App 的 Client Secret
- `repo`：GitHub 上的仓库名字，用于存放 Gitalk 评论
- `owner`：GitHub 仓库的所有者的名字
- `admin`：GitHub 仓库的所有者和合作者 (对这个 Repository 有写权限的用户)
- `id`：这里一个 id 对应 GitHub 仓库的一个 issue，这里是接下来的关键

之前，整个博客站点只对应一个 issue 的问题根源就在于，整个网页只有一个唯一 `id` 的 `Gitalk` 实体，所以这里就需要改成每一篇文章都要对应一个 `Gitalk` 实体，而这里的实现方式就是将每一篇文章对应的 `url` 进行编码，让其结果作为 `Gitalk` 实体的 `id` 这样就实现了一篇文章对应一个 `url` 也对应一个 `Gitalk` 实体同时也对应 GitHub 仓库中的一个 issue。这样就完成了当初既定的功能，每一篇文章的评论都是独立的。下方将附上代码，这种才是真正的配置方式：

```html
......
<script>
    let gitalk = new Gitalk({
      clientID: '25ab86a6c25ef747b6c3',
      clientSecret: '4adf86eee816b4557731dceb18fb5314463c6665',
      repo: 'LauGaHo.github.io',
      owner: 'LauGaHo',
      admin: ['LauGaHo'],
      id: decodeURI(window.location.hash),      // Ensure uniqueness and length less than 50
      distractionFreeMode: false  // Facebook-like distraction free mode
    });

    // 检测到 hash 值发生变化后，直接重新创建一个 gitalk
    window.onhashchange = function (e) {
      gitalk = new Gitalk({
        clientID: '25ab86a6c25ef747b6c3',
        clientSecret: '4adf86eee816b4557731dceb18fb5314463c6665',
        repo: 'LauGaHo.github.io',
        owner: 'LauGaHo',
        admin: ['LauGaHo'],
        id: decodeURI(window.location.hash),      // Ensure uniqueness and length less than 50
        distractionFreeMode: false  // Facebook-like distraction free mode
      });
    }
  </script>
```

因为 Docsify 默认使用了 hash 的路由方式，所以我们也沿用 hash 的路由方式。因为 history 的路由方式在没有任何配置的情况下，遇到了刷新的状况就会显示 404。为了省事就使用了 hash 路由方式。这里的话，我们对浏览器的 `window.onhashchange` 事件进行一个监听的操作，如果监听到了 `url` 中的 `hash` 发生了变化之后，就会对 `hash` 进行一个编码，然后将编码的结果作为 `id` 来新建一个新的 Gitalk 实体。这样新的一个 Gitalk 实体就会对应一个新的 issue。这里问题的关键就在于 `id`。监听 `onhashchange` 就是为了更新 `id`，更新 Gitalk 实体。明白这样即可。