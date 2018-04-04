# 第六章

## 使用Vuex进行状态管理

在这一章之前，我们所有数据都存储在组件（components）中。我们调用一个API，并将返回的数据存储在数据对象上。我们将表单绑定到一个对象，并将该对象存储在数据对象上。组件之间的所有通信都是使用events（从子组件到父组件）和props（从父组件到子组件）完成的。相对于比较简单的应用它可以良好运行，但对于更复杂的应用程序来说，还是不够的。

让我们来看一个社交网络应用 —— 特别是消息通知的场景。你希望顶部导航中的图标显示当前消息的数量，同时你也希望在页面底部的弹窗中也弹出消息的数量。这两个组件在页面上互不相邻，因此使用events和props链接它们将是一场噩梦：与消息通知完全无关的组件将不得不感知到由它们传递的和它们无关的events。有一种替代方案，不将它们链接在一起以共享数据，而是从每个组件分别发出API请求，而这样可能会更糟！每个组件都会在不同的时间进行更新，这意味着它们可能会显示不同的内容，并且页面会发出比实际所需更多的API请求。

 *vuex*是一个帮助开发人员在Vue应用程序中管理应用程序状态的库。它提供了一个集中式store，你可以在整个应用程序中使用这些store来存储和使用全局状态，这使你能够对传入的数据进行验证，以确保再次输出的数据是可预测且正确的。

### 安装

你可以通过使用CDN来使用vuex。只需要添加以下内容：

     <script src="https://unpkg.com/vuex"></script>

另外，如果你使用的是npm，则可以使用 `npm install --save vuex` 来安装vuex。如果你使用的是webpack之类的打包工具，那么就像使用`vue-router`一样，你必须调用`Vue.use()`：

    import Vue from 'vue';
    import Vuex from 'vuex';

    Vue.use(Vuex);

然后你需要创建你的store。编写如下代码并将其保存为 *store/index.js*：

    import Vuex from 'vuex';

    export default new Vuex.Store({
      state: {}
    });

现在，这只是一个空的store：我们将在整个章节中使用它来存储数据。

然后，将它导入到主应用程序文件中，并在创建Vue实例时将其添加为一个属性：

    import Vue from 'vue';
    import store from './store';

    new Vue({
      el: '#app',
      store,
      components: {
        App
      }
    });

你现在已将store添加到应用程序中了，可以通过 `this.$store` 来访问它。接下来让我们先了解一下vuex的概念，然后再看看我们可以用 `this.$store` 做些什么。

### 概念

正如本章导言中所提到的，当一个复杂应用中的多个组件需要共享状态时，vuex是必需的。

让我们用一个没有使用vuex编写的简单组件在页面上显示用户的消息数量：

    const NotificationCount = {
      template: `<p>Messages: {{ messageCount }}</p>`,
      data: () => ({
        messageCount: 'loading'
      }),
      mounted() {
        const ws = new WebSocket('/api/messages');

        ws.addEventListener('message', (e) => {
          const data = JSON.parse(e.data);
          this.messageCount = data.messages.length;
        });
      }
    };

代码很简单。它建立了一个到*/api/messages*的websocket，然后当服务器向客户端发送数据时 —— 在这个例子中，当socket套接字被打开时（初始消息数量）以及消息数量更新时（接收到新消息） —— 通过socket套接字发送的消息将被计数并显示在页面上。

<img src='./images/1.png' style='float:left;margin-right:20px'/> 

>实际应用中，情况会更加复杂一些：在这个例子中，websocket连接并上没有认证环节，通过websocket的响应也假定是有效的JSON，并且其message属性被设定为一个数组，但实际情况中可能并不是这样。对于这个例子，这些简单的代码将完成这项工作。

当我们想在同一页面上使用多个Notification Count组件时，我们就会发现问题。由于每个组件都建立了一个WebSocket，这样就会产生一些不必要的重复连接，并且由于网络延迟，组件的更新时间也可能会稍有不同。为了解决这个问题，我们可以将WebSocket的逻辑转移到vuex中。

马上来看一个例子，我们的组件将变成下面这样：

    const NotificationCount = {
      template: `<p>Messages: {{ messageCount }}</p>`, computed: {
        messageCount() {
          return this.$store.state.messages.length;
        }
      }
      mounted() {
        this.$store.dispatch('getMessages');
      }
    };

以下是我们的vuex store：

    let ws;
    export default new Vuex.Store({
      state: {
        messages: [],
      },
      mutations: {
        setMessages(state, messages) {
          state.messages = messages;
        }
      },
      actions: {
        getMessages({ commit }) {
          if (ws) {
            return;
          }
          ws = new WebSocket('/api/messages');
          ws.addEventListener('message', (e) => {
            const data = JSON.parse(e.data); commit('setMessages', data.messages);
          });
        }
      }
    });

现在，每个已挂载的计数通知组件都将触发getMessages，该操作会检查websocket是否存在，并且仅在没有已打开的连接时才打开连接。然后它监听socket套接字，对状态（state）进行更改，然后计数通知组件就会更新状态，因为store是响应式的 —— 就像Vue中的大多数其他事情一样。当socket套接字发送新的东西时，全局store将被更新，并且页面上的每个组件也同时更新。

在本章节的其余部分中，我将会介绍在这个例子中看到的各个概念 —— 状态（state），变异（mutations），行为（actions） —— 并介绍一种方法，使我们可以在大型应用程序中构造我们的vuex模块, 以避免出现一个大而凌乱的文件。

### State和State Helpers

首先，我们来看看state。状态（State）表示数据如何存储在我们的`vuex store`中。它就像一个我们可以从程序的任何地方访问到的大对象 —— *这是真相的唯一来源*。

我们来看一个只包含一个数字的简单store：

    import Vuex from 'vuex';
    export default new Vuex.Store({
      state: {
        messageCount: 10
      }
    });
