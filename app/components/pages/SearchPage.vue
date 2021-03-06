<template>
  <div class="search-page-container" :class="showArticles ? 'article' : 'user'" @scroll="infiniteScroll">
    <app-header showDefaultHeaderNav showOnlySessionLinks class="logo-original"/>
    <h1 class="area-title">{{ title }}</h1>
    <form @submit.prevent="search" class="area-search">
      <input
        type="text"
        class="form-input"
        required
        maxlength="150"
        v-model.trim="inputText"
        ref="searchInput">
      <img
        @click="search"
        class="search-icon"
        src="~assets/images/pc/common/icon_search.png">
    </form>
    <nav class="area-nav" v-if="showNav">
      <nuxt-link
        :to="{ path: '/search', query: { context: 'article', q: this.query }}"
        class="area-article nav-link"
        :class="{ 'selected': showArticles }">記事</nuxt-link>
      <nuxt-link
        :to="{ path: '/search', query: { context: 'user', q: this.query }}"
        class="area-user nav-link"
        :class="{ 'selected': !showArticles }">ユーザー</nuxt-link>
    </nav>
    <div class="area-search-result">
      <no-ssr>
        <div v-if="showArticles">
          <p class="no-result-message" v-if="searchArticles.articles.length === 0">
           {{  searchArticles.isFetching || !showNav ? '' : '該当する検索結果が存在しません。'}}
          </p>
          <search-article-card-list :articles="searchArticles.articles" v-else/>
        </div>
        <div v-else>
          <p class="no-result-message" v-if="searchUsers.users.length === 0">
           {{ searchUsers.isFetching || !showNav ? '' : '該当する検索結果が存在しません。'}}
          </p>
          <search-user-card-list :users="searchUsers.users" v-else/>
        </div>
      </no-ssr>
    </div>
    <the-loader :isLastPage="!this.query || searchArticles.isLastPage" v-if="showArticles"/>
    <the-loader :isLastPage="!this.query || searchUsers.isLastPage" v-else/>
    <app-footer/>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import { ADD_TOAST_MESSAGE } from 'vuex-toast'
import AppHeader from '../organisms/AppHeader'
import SearchArticleCardList from '../organisms/SearchArticleCardList'
import SearchUserCardList from '../organisms/SearchUserCardList'
import TheLoader from '../atoms/TheLoader'
import AppFooter from '../organisms/AppFooter'

export default {
  components: {
    AppHeader,
    SearchArticleCardList,
    SearchUserCardList,
    TheLoader,
    AppFooter
  },
  computed: {
    title() {
      return 'SEARCH'
    },
    ...mapGetters('article', ['searchArticles']),
    ...mapGetters('user', ['searchUsers']),
    ...mapGetters('presentation', ['searchArticlesScrollHeight', 'searchUsersScrollHeight'])
  },
  created() {
    this.showArticles = this.$route.query.context !== 'user'
    this.query = this.$route.query.q
    this.showNav = !!this.query
  },
  async mounted() {
    if (window.innerWidth > 640) this.$refs.searchInput.focus()
    this.inputText = this.$route.query.q
    await this.$nextTick()
    if (this.searchArticlesScrollHeight) this.$el.scrollTop = this.searchArticlesScrollHeight
    if (this.searchUsersScrollHeight) this.$el.scrollTop = this.searchUsersScrollHeight
  },
  data() {
    return {
      isFetchingData: false,
      showArticles: true,
      query: null,
      showNav: false,
      inputText: '',
      isSearchFirstly: true
    }
  },
  beforeDestroy() {
    this.showArticles
      ? this.setSearchArticlesScrollHeight({ scrollHeight: this.$el.scrollTop })
      : this.setSearchUsersScrollHeight({ scrollHeight: this.$el.scrollTop })
  },
  methods: {
    async search() {
      try {
        if (window.innerWidth <= 640) document.activeElement.blur()
        this.resetSearchData()
        this.query = this.inputText
        if (this.isFetchingData || !this.query) return
        this.isFetchingData = true
        this.showNav = true
        const path = `/search?context=${this.showArticles ? 'article' : 'user'}&q=${this.query}`
        this.isSearchFirstly ? this.$router.replace(path) : this.$router.push(path)
        this.isSearchFirstly = false
        await this.getSearchData(this.query)
      } catch (error) {
        this.ADD_TOAST_MESSAGE({ text: '検索結果の取得に失敗しました。', type: 'warning' })
        console.error(error)
      } finally {
        this.isFetchingData = false
      }
    },
    async getSearchData(query) {
      this.showArticles
        ? await this.getSearchArticles({ query })
        : await this.getSearchUsers({ query })
    },
    async infiniteScroll(event) {
      if (this.isFetchingData || !this.query) return
      try {
        this.isFetchingData = true

        const isLastPage = this.showArticles
          ? this.searchArticles.isLastPage
          : this.searchUsers.isLastPage
        const isScrollBottom =
          event.target.scrollTop + event.target.offsetHeight >= event.target.scrollHeight - 10
        if (isLastPage || !isScrollBottom) return

        await this.getSearchData(this.query)
      } finally {
        this.isFetchingData = false
      }
    },
    resetSearchData() {
      this.resetSearchArticles()
      this.resetSearchArticlesPage()
      this.resetSearchArticlesIsLastPage()
      this.resetSearchUsers()
      this.resetSearchUsersPage()
      this.resetSearchUsersIsLastPage()
    },
    async fetchSearchedData(query) {
      this.resetSearchData()
      this.query = query
      if (typeof this.query !== 'string') return
      this.inputText = this.query
      await this.getSearchData(this.query)
    },
    ...mapActions({ ADD_TOAST_MESSAGE }),
    ...mapActions('user', [
      'getSearchUsers',
      'resetSearchUsers',
      'resetSearchUsersPage',
      'resetSearchUsersIsLastPage'
    ]),
    ...mapActions('article', [
      'getSearchArticles',
      'resetSearchArticles',
      'resetSearchArticlesPage',
      'resetSearchArticlesIsLastPage'
    ]),
    ...mapActions('presentation', ['setSearchArticlesScrollHeight', 'setSearchUsersScrollHeight'])
  },
  watch: {
    $route(to, from) {
      const { query } = to
      this.showArticles = query.context !== 'user'
      this.showNav = !!query.q
      this.isSearchFirstly = !this.showNav
      this.fetchSearchedData(query.q)
    }
  }
}
</script>

<style lang="scss" scoped>
.search-page-container {
  background-size: contain;
  display: grid;
  grid-row-gap: 20px;
  grid-template-rows: 100px 60px 20px 80px 1fr 75px 75px;
  /* prettier-ignore */
  grid-template-areas:
    "app-header  app-header             app-header"
    "...         title                  ...       "
    "...         search                 ...       "
    "...         nav                    ...       "
    "...         search-result          ...       "
    "...         loader                 ...       "
    "app-footer  app-footer             app-footer";
  height: 100vh;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;

  &.article {
    grid-template-columns: 1fr 1080px 1fr;
  }

  &.user {
    grid-template-columns: 1fr 640px 1fr;
  }
}

.area-title {
  font-family: 'Times New Roman', Times, serif;
  text-align: center;
  font-size: 20px;
  margin: 0;
  grid-area: title;
  letter-spacing: 0.2em;
}

.area-search {
  grid-area: search;
  margin: 0 auto;
  position: relative;

  .form-input {
    -webkit-appearance: none;
    border: none;
    border-radius: 0;
    border-bottom: 1px dotted #232538;
    padding: 5px 24px 5px 0;
    width: 400px;

    &:focus {
      outline: 0;
    }
  }

  .search-icon {
    position: absolute;
    right: 0;
    top: 6px;
    width: 16px;
    cursor: pointer;
  }
}

.area-nav {
  grid-area: nav;
  display: grid;
  text-align: center;
  grid-template-rows: 1fr 30px 1fr;
  grid-template-columns: 1fr 70px 70px 1fr;
  grid-column-gap: 10px;
  /* prettier-ignore */
  grid-template-areas:
    "... ...     ...  ..."
    "... article user ..."
    "... ...     ...  ...";

  .nav-link {
    font-size: 14px;
    text-decoration: none;
    color: #525256;
    cursor: pointer;

    &.selected {
      color: #99a2ff;
      border-bottom: 2px solid #99a2ff;
    }
  }

  .area-article {
    grid-area: article;
  }

  .area-user {
    grid-area: user;
  }
}

.area-search-result {
  grid-area: search-result;
}

.no-result-message {
  text-align: center;
}

@media screen and (max-width: 1296px) {
  .search-page-container {
    &.article {
      grid-template-columns: 1fr 710px 1fr;
    }
  }
}

@media screen and (max-width: 920px) {
  .search-page-container {
    &.article {
      grid-template-columns: 1fr 340px 1fr;
    }

    &.user {
      grid-template-columns: 1fr 70vw 1fr;

      .area-search {
        width: 340px;
      }
    }
  }

  .area-search {
    width: 100%;

    .form-input {
      width: calc(100% - 24px);
    }
  }
}

@media screen and (max-width: 640px) {
  .search-page-container {
    grid-template-columns: 1fr 340px 1fr;
    grid-template-rows: 100px 40px 20px 80px 1fr 75px min-content;
  }

  .area-title {
    font-size: 16px;
  }
}

@media screen and (max-width: 550px) {
  .search-page-container {
    grid-template-rows: 60px 20px 20px 50px 1fr 75px;
    /* prettier-ignore */
    grid-template-areas:
    "app-header  app-header             app-header"
    "...         title                  ...       "
    "...         search                 ...       "
    "nav         nav                    nav       "
    "...         search-result          ...       "
    "...         loader                 ...       "
    "app-footer  app-footer             app-footer";

    &.user {
      grid-template-columns: 10px 1fr 10px;
    }
  }
}

@media screen and (max-width: 370px) {
  .search-page-container {
    grid-template-columns: 10px 1fr 10px;

    &.article {
      grid-template-columns: 10px 1fr 10px;
    }

    &.user {
      .area-search {
        width: 100%;
      }
    }
  }
}
</style>
