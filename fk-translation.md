# 第六章

## 使用Vuex做状态管理

在本书的这一章之前，我们所有数据都已存储在组件中。我们调用一个API，并将返回的数据存储在数据对象上。我们将表单绑定到一个对象，并将该对象存储在数据对象上。组件之间的所有通信都是使用events（从子类到父类）和props（从父类到子类）完成的。简单的情况下它可以良好运行，但对于更复杂的应用程序来说，这是不够的。

让我们来看一个社交网络应用 - 尤其是消息。你希望顶部导航中的图标显示你拥有的消息数量，然后你需要在页面底部弹出消息，这些消息也会告诉你消息的数量。这两个组件在页面上都不相邻，因此使用events和props链接它们将是一场噩梦：与消息通知完全无关的组件将不得不感知到由它们传递的和它们无关的events。有一种替代方法，不将它们链接在一起以共享数据，而是从每个组件分别发出API请求，那样会更糟！每个组件都会在不同的时间进行更新，这意味着它们可能会显示不同的内容，并且页面会发出比实际所需更多的API请求。

*vuex*是一个帮助开发人员在Vue应用程序中管理应用程序状态的库。它提供了一个集中式store，您可以在整个应用程序中使用这些store来存储和使用全局状态，使您能够验证数据，以确保再次输出的数据具有可预测性和正确性。

### 安装

你可以通过使用CDN来加载vuex。只需添加以下内容：

     <script src="https://unpkg.com/vuex"></script>

或者，如果你使用npm，则可以使用 `npm install --save vuex` 来安装vuex。如果你使用的是webpack之类的打包工具，那么就像使用vue-router一样，你必须调用`Vue.use()`：

    import Vue from 'vue';
    import Vuex from 'vuex';
    Vue.use(Vuex);

然后你需要创建你的store。我们创建以下文件并将其保存为 *store/index.js*：

    import Vuex from 'vuex';

    export default new Vuex.Store({
      state: {}
    });

现在，这只是一个空的store：我们将在整个章节中添加它。

然后，将它导入到主应用程序文件中，并在创建Vue实例时将其添加为属性：

    import Vue from 'vue';
    import store from './store';

    new Vue({
      el: '#app',
      store,
      components: {
        App
      }
    });

你现在已store添加到应用程序中，你可以通过 `this.$store` 来访问它。让我们看看vuex的概念，然后看看我们可以用 `this.$store` 可以做些什么。

### 概念

正如本章导言中所提到的，当复杂应用程序需要多个组件来共享状态时，vuex是必需的。

让我们用一个没有使用vuex编写的简单组件来展示用户在页面上显示的消息数量：

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

这很简单。它打开一个到*/api/messages*的websocket，然后当服务器向客户端发送数据时 — 在这种情况下，当套接字打开时（初始消息数量）以及计数更新时（在新消息上） - 通过套接字发送的消息将被计数并显示在页面上。

<img src='./images/1.png' style='float:left;margin-right:20px'/>

实际应用中，这段代码会更加复杂：在这个例子中，websocket上没有认证，并且总是认为通过websocket的响应是有效的JSON，其message属性是一个数组，但实际上它可能不是。对于这个例子，这些简单的代码将完成这项工作。

当我们想在同一页面上使用多个Notification Count组件时，我们遇到了问题。由于每个组件 建立了一个WebSocket，它会建立不必要的重复连接，并且由于网络延迟，组件可能会在稍微不同的时间更新。为了解决这个问题，我们可以将WebSocket逻辑转移到vuex中。

让我们来看一个例子吧，我们的组件将变成这样：

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

现在，每个挂载的计数通知组件都将触发getMessages，该操作会检查websocket是否存在，并且仅在没有打开连接时才打开连接。然后它监听套接字，对state进行更改，然后在计数通知组件中更新状态，因为store是响应式的 - 就像Vue中的大多数其他事情一样。当套接字发送新的东西时，全局store将被更新，并且页面上的每个组件都将同时更新。

在本章节的其余部分中，我将介绍给你在该示例中看到的单个概念 - state，mutations和actions - 并说明一种方法，使我们可以在大型应用程序中构造我们的vuex模块, 以避免出现一个大而凌乱的文件。

### State和State Helpers

首先, 让我们看看state。State表明数据在我们的vuex存储中的存储方式。这就像一个大对象, 我们可以从任何地方在我们的应用程序-这是唯一的真理来源。

首先，我们来看看state。State表示数据如何存储在我们的vuex store中。这就像一个我们可以从应用程序的任何地方访问的大对象 - *这是真相的唯一来源*。

我们来看一个只包含一个数字的简单store：

    import Vuex from 'vuex';
    export default new Vuex.Store({
      state: {
        messageCount: 10
      }
    });
