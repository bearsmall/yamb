<template>
  <div class="main">
    <div class="page-loading loader-inner ball-pulse" v-if="isLoading">
      <div> </div> <div> </div> <div> </div>
    </div>
    <section class="thread" v-show="! isLoading">
      <div class="container">
        <div class="thread-header" v-show="currentPage == 1">
          <h2>{{ title }}</h2>
          <div class="columns is-mobile">
            <div class="column is-7">
              <small>{{ time }}</small>
            </div>
            <div class="column">
              <a @click="jumpToBoard(board.name)"><small>{{ board.description + '/' + board.name }}</small></a>
            </div>
          </div>
          <hr>
        </div>

        <div v-if="mainPost && currentPage == 1" class="article">
          <div class="poster media">
            <figure class="media-left">
              <p class="image is-32x32"> <img :src="mainPost.poster.face_url"> </p>
            </figure>
            <div class="media-content">
              <div class="content">
                <h4> {{ mainPost.poster.id }}</h4>
                <small> {{ mainPost.time }} </small>
              </div>
            </div>
            <div class="media-right">
              <span>楼主</span>
            </div>
          </div>
          <div class="article-body content"  v-html="mainPost.content"> </div>
        </div>
        <div class="oops" v-else-if="! mainPost">
          // TODO threads without header
          <br>
          // CONTRIBUTING: <a href="https://github.com/paper777/yamb">Go To Github Page</a>
          <hr>
        </div>


        <div class="popular-replies" v-if="currentPage == 1 && popularReplies.length">
          <div class="reply-tag">
            <span class="tag is-danger">精彩回复</span> 
            <b></b>
          </div>
          <div class="post" v-for="(article, index) in popularReplies" :key="index">
            <div class="poster media">
              <figure class="media-left">
                <p class="image is-32x32"> <img :src="article.poster ? article.poster.face_url : ''"> </p>
              </figure>
              <div class="media-content">
                <div class="content">
                  <h4> {{ article.poster ? article.poster.id : '已注销'}}</h4>
                  <small> {{ article.time }}</small>
                </div>
              </div>
              <div class="media-right">
                <span>{{ article.pos == 1 ? '沙发' : (article.pos == 2 ? '板凳' : article.pos + '楼') }}</span>
                <span>
                  <a v-if="! article.voted" @click="voteup(article, index)">
                    <i class="iconfont icon-voteup"></i>
                  </a>
                  <i v-else class="iconfont icon-voteup"></i>
                  {{ article.voteup_count }}
                </span>
              </div>
            </div>
            <div class="article-body content" v-html="article.content"> </div>
            <hr v-if="index < popularReplies.length - 1">
          </div>
        </div>

        <div class="posts">
          <div class="reply-tag" v-if="currentPage == 1 && posts.length">
            <span class="tag is-primary">全部回复</span>
            <b></b>
          </div>
          <div class="post" v-for="(article, index) in posts" :key="index">
            <div class="poster media">
              <figure class="media-left">
                <p class="image is-32x32"> <img :src="article.poster ? article.poster.face_url : ''"> </p>
              </figure>
              <div class="media-content">
                <div class="content">
                  <h4> {{ article.poster ? article.poster.id : '已注销'}}</h4>
                  <small> {{ article.time }}</small>
                </div>
              </div>
              <div class="media-right">
                <span>{{ article.pos == 1 ? '沙发' : article.pos == 2 ? '板凳' : article.pos +  '楼' }}</span>
                <span>
                  <a v-if="! article.voted" @click="voteup(article, index)">
                    <i class="iconfont icon-voteup"></i>
                  </a>
                  <i v-else class="iconfont icon-voteup"></i>
                  {{ article.voteup_count }}
                </span>
              </div>
            </div>
            <div class="article-body content" v-html="article.content"> </div>
            <hr>
          </div>
        </div>
      </div>
    </section>
    <section class="paginate" v-if="! isLoading && totalPage > 1">
      <div class="card">
        <header class="columns is-mobile paginate-items">
          <div class="column">
            <a v-if="mainPost && ! mainPost.voted" @click="voteup(mainPost, -1)">顶 {{ mainPost.voteup_count }}</a>
            <a v-else>顶 {{ mainPost.voteup_count }}</a>
          </div>
          <div class="column">
            <a @click="getPrevPage">上一页</a>
          </div>
          <div class="column">
            <a>{{ currentPage + '/' + totalPage }}</a>
          </div>
          <div class="column">
            <a @click="getNextPage">下一页</a>
          </div>
          <div class="column">
            <a @click="getReply">回复</a>
          </div>
        </header>
      </div>
    </section>
  </div>
</template>

<script>
import * as api from 'api/thread'

export default {
  data () {
    return {
      query: {
        board: '',
        id: ''
      },

      isLoading: true,

      title: '加载中...',
      time: '',
      gid: 0,
      board: {
        description: 'loading',
        name: '...'
      },

      // pagination
      currentPage: 1,
      totalPage: 1,

      anony: false,

      // posts
      mainPost: null,
      posts: [],
      popularReplies: [],
      cachePosts: {}
    }
  },

  watch: {
  },

  created() {
    this.query = this.$route.params;
    this.fetchArticles();
  },

  methods: {
    fetchArticles() {
      this.isLoading = true;
      const page = this.currentPage;
      if (page in this.cachePosts) {
        this.isLoading = false;
        return this.posts = this.cachePosts[page];
      }
      api.getThread(this.query.board, this.query.id, { page }).then((res) => {
        if (! res.success) {
          // TODO
          return false;
        }

        const data = res.data;
        if (page == 1) {
          this.title = data.title;
          this.time = data.time;
          this.gid = data.gid;
          this.board = data.board;
          this.anony = data.anony;
          this.popularReplies = data.popularReplies;
          this.totalPage = data.pagination.total;
          if (data.articles[0]['id'] == this.gid) {
            [this.mainPost, ...this.posts] = data.articles;
          } else {
            this.posts = data.articles;
          }
        } else {
          this.posts = data.articles;
        }
        this.cachePosts[page] = this.posts;
        document.body.scrollTop = document.documentElement.scrollTop = 0;
        this.isLoading = false;
      });
    },

    jumpToBoard(name) {
      this.$router.push('/board/' + name);
    },

    getPrevPage() {
      if (this.currentPage <= 1) {
        return false;
      }
      this.currentPage --;
      this.fetchArticles();
    },

    getNextPage() {
      if (this.currentPage >= this.totalPage) {
        return false;
      }
      this.currentPage ++;
      this.fetchArticles();
    },

    voteup(article, index) {
      api.voteup(this.board.name, { id: article.id }).then((res) => {
        if (! res.success) {
          // TODO 
          return false;
        }
        article.voted = true;
        article.voteup_count = res.data.count;
        if (index == -1) {
          return this.mainPost = article;
        }
        this.posts[index] = article;
      });
    },

    getReply() {
    }

  }
}
</script>

<style scoped>
h4 {
    margin: 0;
}
.container {
    padding: 12px 12px;
    background-color: #fff;
}
.poster.media {
    margin: 8px 0;
}
.article-body.content {
    color: black;
}

.tread-header {
    height: 60px;
}
.paginate {
    text-align: center;
    margin: 1px;
}
.paginate-items {
  margin-left: 0;
  margin-right: 0;
}
.reply-tag {
    margin: 16px 0;
}
.reply-tag b {
    display: inline-block;
    height: 1px;
    width: 79%;
    background: #ddd;
    float: right;
    margin-top: 13px;
}
</style>
