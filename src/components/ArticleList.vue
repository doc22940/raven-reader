<template>
  <div class="articles-list">
    <div class="articles-inner">
      <form class="search-form">
        <div class="search-input input-group mb-0">
          <div class="input-group-prepend">
            <span class="input-group-text">
              <feather-icon name="search" />
            </span>
          </div>
          <input
            v-model.lazy="search"
            type="text"
            class="form-control"
            @change="searchList"
            placeholder="Search"
            aria-label="Search"
          >
          <div class="toolsbar">
            <div class="tool">
              <button
                v-b-tooltip.hover
                class="btn btn-toolbar"
                type="button"
                title="Refresh All Feeds"
                aria-label="Refresh All Feeds"
                @click="sync"
              >
                <feather-icon
                  name="refresh-cw"
                  :class="{ 'fa-spin': syncState }"
                />
              </button>
            </div>
            <div class="tool">
              <button
                ref="markallread"
                v-b-tooltip.hover
                v-b-modal.markallread
                class="btn btn-toolbar"
                type="button"
                title="Mark all as read"
              >
                <feather-icon name="check-circle" />
              </button>
            </div>
          </div>
        </div>
      </form>
      <div class="articles">
        <perfect-scrollbar class="list-group">
          <template v-if="filteredArticles.length > 0">
            <template v-for="(article, index) in mapArticles(filteredArticles)">
              <article-item
                v-if="index <= articlesToShow"
                :key="article.uuid"
                :article="article"
              />
            </template>
            <button v-if="articlesToShow <= mapArticles(filteredArticles).length" @click="articlesToShow += 10" class="btn btn-primary rounded-0" type="button">
              Load more
            </button>
          </template>
          <div
            v-if="filteredArticles.length === 0"
            class="no-articles"
          >
            No articles available
          </div>
        </perfect-scrollbar>
      </div>
      <div
        ref="statusBar"
        class="statusBar"
      >
        <button
          class="btn foldBtn"
          type="button"
          :aria-label="ariaLabelFoldSidebar"
          @click="fold"
        >
          <feather-icon :name="featherIcon" />
        </button>
        <span
          ref="statusMsg"
          class="statusMsg"
        />
      </div>
    </div>
  </div>
</template>
<script>
import { mapGetters } from 'vuex'
import helper from '../services/helpers'
import bus from '../services/bus'

export default {
  props: {
    type: {
      type: String,
      default: 'all'
    },
    feed: {
      type: String,
      default: null
    }
  },
  data () {
    return {
      articlesToShow: 24,
      search: null,
      featherIcon: 'chevron-left',
      ariaLabelFoldSidebar: 'Hide Sidebar',
      syncState: false
    }
  },
  mounted () {
    bus.$on('reset-articlelist-count', () => {
      this.articlesToShow = 24
    })
    bus.$on('sync-complete', () => {
      this.itemsChange()
    })
  },
  computed: {
    ...mapGetters([
      'filteredArticles'
    ]),
    activeArticleId () {
      return this.$store.getters.activeArticleId
    }
  },
  watch: {
    filteredArticles: 'itemsChange'
  },
  methods: {
    searchList () {
      this.$store.dispatch('changeType', {
        type: 'search',
        search: this.search,
        feed: null,
        category: null
      })
    },
    mapArticles (articles) {
      return articles.map(article => ({ ...article, id: article.articles.id, isActive: this.isArticleActive(article) }))
    },
    isArticleActive (article) {
      return !!article && article.articles.id !== undefined && article.articles.id === this.activeArticleId
    },
    itemsChange () {
      if (this.filteredArticles) {
        this.$refs.statusMsg.innerText = `${this.filteredArticles.length} items`
      }
    },
    sync () {
      this.syncState = true
      if (this.$store.state.Feed.feeds.length > 0) {
        this.$parent.$refs.topProgress.start()
        window.log.info(`Processing ${this.$store.state.Feed.feeds.length} feeds`)
        this.$refs.statusMsg.innerText = 'Syncing...'
        helper.subscribe(this.$store.state.Feed.feeds, null, true, false).then(() => {
          this.itemsChange()
          this.$parent.$refs.topProgress.done()
          this.syncState = false
        })
      }
    },
    fold () {
      this.$parent.$refs.sidebar.hidden = !this.$parent.$refs.sidebar.hidden
      this.featherIcon = this.$parent.$refs.sidebar.hidden ? 'chevron-right' : 'chevron-left'
      this.ariaLabelFoldSidebar = this.$parent.$refs.sidebar.hidden ? 'Show Sidebar' : 'Hide Sidebar'
    }
  }
}
</script>
<style lang="scss">
.articles-inner {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow:hidden;
}

// Default color mode
:root {
  & .articles-list {
    --input-background-color: var(--background-color);
    --input-color: #495057;
  }
}

// Dark color mode
.app-sunsetmode {
  &.articles-list {
    --input-background-color: var(--background-color);
    --input-color: #c8cacc;
  }
  .search-input {
    input[type="text"] {
      &:focus {
        background: var(--background-color);
        color: #000;
      }
    }
  }
}
.app-nightmode {
  & .articles-list {
    --input-background-color: var(--background-color);
    --input-color: #c8cacc;
  }

  .search-input {
    input[type="text"] {
      &:focus {
        background: var(--background-color);
        color: white;
      }
    }
  }
}
.app-darkmode {
  & .articles-list {
    --input-background-color: var(--background-color);
    --input-color: #c8cacc;
  }

  .search-input {
    input[type="text"] {
      &:focus {
        background: var(--background-color);
        color: white;
      }
    }
  }
}

.articles-list {
  position: relative;
  flex-grow: 0;
  font-size: 14px;
  width: 350px;
  height: 100%;
  border-right: 1px solid var(--border-color);

  &:after {
    content: '';
    position: absolute;
    bottom: 0;
    width: 100%;
    height: 25px;
    background: linear-gradient(rgba(255, 255, 255, 0.001), white ); /* transparent keyword is broken in Safari */
    pointer-events: none;
  }

  .search-form {
    border-bottom-color: var(--border-color);

    .form-control {
      background: var(--input-background-color);
      color: var(--input-color);
    }
  }
  .articles-inner .search-form,
  .articles-inner .articles,
  .articles-inner .list-group-item {
    background: var(--background-color);
    color: var(--text-color);
  }
  .articles-inner .list-group-item {
    background: var(--background-color);
    border-left: 0;
    border-right: 0;
    border-bottom-color: var(--border-color);
  }

  &::after {
    background: var(--after-background);
  }
}

.articles {
  position: absolute;
  top: 41px;
  bottom: 0;
  left: 0;
  right: 0;
  overflow-y: auto;
  background: #fff;
  overflow-x: hidden;
  height: calc(100% - 80px);
}

.no-articles {
  display: flex;
  height: 90vh;
  justify-content: center;
  align-items: center;
}

.article-read {
  opacity: 0.7;
}

.search-input {
  position: absolute;
  top: 0;
  bottom: 0;

  color: var(--text-color);

  .feather {
    color: var(--text-color);
  }

  .form-control::placeholder {
    color: var(--text-color);
  }

  .input-group-text {
    background: none;
    border: 0;
  }
}

.search-form {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  margin: 0;
  border-bottom: 1px solid transparent;
  border-color: #dcdee0;
  background-color: #FFFFFF;
  height: 41px;

  .form-control {
    border-radius: 0;
    border: 0;

    &:focus {
      box-shadow: none;
      outline: 0;
    }
  }
}

.statusBar {
  color: var(--text-color);
  height: 40px;
  line-height: 30px;
  width: 100%;
  background-color: var(--background-color);
  position: absolute;
  bottom: 0;
  border-top: 1px solid var(--border-color);
  display: flex;
  align-items: center;
  z-index: 20;

  & .feather {
    color: var(--text-color);
  }
}

.statusMsg {
  line-height: 30px;
  width: 100%;
  font-size: 12px;
  text-align: center;
}

.foldBtn {
  left: 0;
  margin: 0 !important;
  padding: 0 !important;
}

.foldBtn:hover {
  color: #3399FF !important;
}

.foldBtn:focus {
  outline: none !important;
  box-shadow: none !important;
}

.foldBtn:active {
  outline: none !important;
  box-shadow: none !important;
}

.list-group {
  height: 100%;
}
.fa-spin {
  -webkit-animation: fa-spin 2s infinite linear;
  animation: fa-spin 2s infinite linear;
}
.fa-pulse {
  -webkit-animation: fa-spin 1s infinite steps(8);
  animation: fa-spin 1s infinite steps(8);
}
@-webkit-keyframes fa-spin {
  0% {
    -webkit-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(359deg);
    transform: rotate(359deg);
  }
}
@keyframes fa-spin {
  0% {
    -webkit-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  100% {
    -webkit-transform: rotate(359deg);
    transform: rotate(359deg);
  }
}

.toolsbar {
  position: relative;
  right: 0;
  top: 0;
  margin: 0;
  height: 41px;
  display: flex;
  z-index: 10;
  align-items: center;

  .btn {
      color: rgb(65, 65, 65);
      margin-right: 2px;
      padding: 0 !important;
      width: 28px;
      box-shadow: none;
      outline: none;
      height: 28px;
      z-index: 10;
      .fill {
        fill: rgb(65, 65, 65);
      }
  }
  .btn:hover {
    color: #007bff;
    .fill {
      fill: #007bff;
    }
  }
}
</style>
