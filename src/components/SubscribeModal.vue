<template>
  <b-modal
    id="addfeed"
    ref="addFeedModal"
    hide-header
    :hide-footer="feeddata === null"
    size="lg"
    centered
    @shown="focusMyElement"
    @hidden="onHidden"
  >
    <form @submit.prevent="fetchFeed">
      <b-input-group size="md">
        <b-input-group-text slot="prepend">
          <strong>
            Add
            <feather-icon name="plus" />
          </strong>
        </b-input-group-text>
        <b-input-group-text slot="append">
          <loader v-if="loading" />
          <div
            v-if="feeddata !== null && !loading"
            class="favicon-img"
          >
            <img
              :src="feeddata.site.favicon"
              height="20"
            >
          </div>
        </b-input-group-text>
        <b-form-input
          ref="focusThis"
          v-model="feed_url"
          class="no-border"
          placeholder="Enter website or feed url"
        />
      </b-input-group>
    </form>
    <div
      v-if="feeddata !== null"
      class="subscription-content col pt-3"
    >
      <template v-for="(feed, index) in feeddata.feedUrls">
        <div :key="index">
          <b-input-group size="md">
            <b-input-group-text slot="append">
              <b-form-checkbox
                v-model="selected_feed"
                :value="feed"
              />
            </b-input-group-text>
            <b-form-input v-model="feed.title" />
          </b-input-group>
          <b-form-text
            id="inputLiveHelp"
            class="mb-3"
          >
            {{ feed.url }}
          </b-form-text>
        </div>
      </template>
      <b-form-select
        v-model="selectedCat"
        v-if="categoryItems.length > 0"
        :options="categoryItems"
        class="mb-3"
      >
        <template slot="first">
          <option :value="null">
            Please select category
          </option>
        </template>
      </b-form-select>
      <p>
        <button
          class="btn btn-link pl-0"
          type="button"
          @click="addCategory"
        >
          Add new category
        </button>
      </p>
      <p v-if="showAddCat">
        <b-form-input
          v-model="newcategory"
          placeholder="Enter new category"
        />
      </p>
    </div>
    <div slot="modal-footer">
      <button
        type="button"
        class="btn btn-secondary mr-2"
        @click="hideModal"
      >
        Close
      </button>
      <button
        type="button"
        class="btn btn-primary"
        :disabled="disableSubscribe"
        @click="subscribe"
      >
        Subscribe
      </button>
    </div>
  </b-modal>
</template>
<script>
import normalizeUrl from 'normalize-url'
import unescape from '../services/unescape'
import helper from '../services/helpers'

export default {
  name: 'AddfeedModal',
  data () {
    return {
      feed_url: '',
      loading: false,
      feeddata: null,
      newcategory: null,
      showAddCat: false,
      selectedCat: null,
      isXML: false,
      selected_feed: []
    }
  },
  computed: {
    categoryItems () {
      return this.$store.state.Category.categories.map((item) => {
        return { value: item.title, text: item.title }
      })
    },
    disableSubscribe () {
      return this.$store.state.Setting.offline
    }
  },
  methods: {
    addCategory () {
      this.showAddCat = !this.showAddCat
    },
    async isContentXML (link) {
      const validHeaders = [
        'application/xml',
        'application/xml; charset=utf-8',
        'application/rss+xml; charset=utf-8',
        'text/xml;charset=UTF-8',
        'text/rss+xml;charset=UTF-8',
        'text/xml; charset=utf-8',
        'text/xml',
        'text/rss+xml',
        'application/rss+xml',
        'application/rdf+xml',
        'application/atom+xml'
      ]
      const content = await window.got(link)
      return validHeaders.includes(content.headers['content-type'])
    },
    async fetchFeed () {
      const self = this
      this.loading = true
      if (!this.$store.state.Setting.offline) {
        if (this.feed_url) {
          try {
            const isXML = await this.isContentXML(normalizeUrl(this.feed_url, { stripWWW: false, removeTrailingSlash: false }))
            console.log(isXML)
            window.rssFinder(normalizeUrl(this.feed_url, { stripWWW: false, removeTrailingSlash: false }), { feedParserOptions: { feedurl: this.feed_url } }).then(
              res => {
                this.loading = false
                res.feedUrls.map(item => {
                  item.title = unescape(item.title)
                  if (isXML) {
                    item.url = self.feed_url
                  }
                  return item
                })
                if (res.feedUrls.length === 0) {
                  this.showError()
                } else {
                  this.selected_feed = []
                  this.selected_feed.push(res.feedUrls[0])
                  this.feeddata = res
                }
              },
              error => {
                if (error) {
                  window.log.info(error)
                }
                this.showError()
              }
            )
          } catch (e) {
            window.log.info(e)
            this.showError()
          }
        } else {
          this.showError()
        }
      } else {
        this.$toasted.show('There is no internet connection', {
          theme: 'outline',
          position: 'top-center',
          duration: 2000
        })
        this.loading = false
      }
    },
    showError () {
      this.loading = false
      this.feeddata = null
      this.$toasted.show('No feed found', {
        theme: 'outline',
        position: 'top-center',
        duration: 2000
      })
    },
    focusMyElement (e) {
      this.$refs.focusThis.focus()
    },
    resetForm () {
      this.feed_url = ''
      this.feeddata = null
      this.url = ''
      this.selected_feed = []
      this.loading = false
      this.newcategory = null
      this.showAddCat = false
      this.selectedCat = null
    },
    hideModal () {
      this.resetForm()
      this.$refs.addFeedModal.hide()
    },
    subscribe () {
      if (this.newcategory === null) {
        this.newcategory = this.selectedCat
      }
      helper.subscribe(this.selected_feed, this.newcategory, false)
      this.hideModal()
    },
    onHidden () {
      this.resetForm()
    }
  }
}
</script>
<style lang="scss">
.app-sunsetmode {
  #addfeed {
    form {
      background: var(--background-color);
      color: var(--text-color);
    }

    .no-border {
      background: var(--background-color);
      color: var(--text-color);
      &:focus {
        color: #000;
      }
    }

    .input-group-text {
      color: #000;
    }

    .subscription-content {
      background: var(--background-color);
      border-color: var(--border-color);
    }
  }
}
.app-nightmode {
  #addfeed {
    form {
      background: #16161d;
      color: var(--text-color);
    }

    .no-border {
      background: #16161d;
      color: var(--text-color);
      &:focus {
        color: #fff;
      }
    }

    .input-group-text {
      color: #fff;
    }

    .subscription-content {
      background: #16161d;
      border-color: var(--border-color);
    }
  }

  .bouncing-loader > div {
    background: #fff;
  }
}
.app-darkmode {
  #addfeed {
    form {
      background: var(--background-color);
      color: var(--text-color);
    }

    .no-border {
      background: var(--background-color);
      color: var(--text-color);
      &:focus {
        color: #fff;
      }
    }

    .input-group-text {
      color: #fff;
    }

    .subscription-content {
      background: var(--background-color);
      border-color: var(--border-color);
    }
  }
}
#addfeed {
  form {
    background: #f4f6f8;
    padding: 0.8rem;
  }

  .modal-body {
    padding: 0rem;
  }
  .input-group-text {
    background: transparent;
    border: 0;
  }
  .no-border {
    border: 0;
    background: #f4f6f8;

    &:focus {
      outline: 0;
      box-shadow: none;
    }
  }
}

@keyframes bouncing-loader {
  from {
    opacity: 1;
    transform: translateY(0);
  }
  to {
    opacity: 0.1;
    transform: translateY(-1rem);
  }
}
.bouncing-loader {
  display: flex;
  justify-content: center;
}
.bouncing-loader > div {
  width: 0.4rem;
  height: 0.4rem;
  margin: 1rem 0rem;
  background: #000;
  border-radius: 50%;
  animation: bouncing-loader 0.6s infinite alternate;
}
.bouncing-loader > div:nth-child(2) {
  animation-delay: 0.2s;
}
.bouncing-loader > div:nth-child(3) {
  animation-delay: 0.4s;
}

.favicon-img {
  display: flex;
  justify-content: center;
}

.subscription-content {
  background: #fff;
  border-top: 1px solid #f4f6f8;
}
</style>
