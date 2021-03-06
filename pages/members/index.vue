<template>
  <div class="members-page">
    <!-- Banner -->
    <section :style="bannerStyle" class="banner">
      <div class="banner-wrapper">
        <div class="text-container">
          <h1 class="banner-title">{{ this.$t('membersPage.bannerTitle') }}</h1>
        </div>
        <div class="image-container" >
          <img :src="bannerMembers" class="front-img">
        </div>
      </div>
    </section>
    <!-- Members -->
    <section class="members-section">
      <div class="members-section-wrapper">
        <Row>
          <!-- Filters -->
          <i-col :xs="{ span: 24 }" :sm="{ span: 6 }" class="filters">
            <MembersFilters :states="states" :loading="filterLoading" @on-filter="filterMembers"/>
          </i-col>
          <!-- List -->
          <i-col :xs="{ span: 24 }" :sm="{ span: 18 }" class="list">
            <Row>
              <i-col v-for="member in members" :key="member.id" span="24">
                <MemberSearchResultCard :member="member" :states="states" />
              </i-col>
            </Row>
            <InfiniteLoading ref="infiniteLoading" @infinite="moreItems">
              <span slot="spinner">
                <Spinner />
              </span>
              <span slot="no-more">
                沒更多資料了 😂
              </span>
              <span slot="no-results">
                沒有更多結果了 😭
              </span>
            </InfiniteLoading>
          </i-col>
        </Row>
      </div>
    </section>
  </div>
</template>

<script>
import appConfig from '~/config/app.json'
// Packages
import _ from 'lodash'
import InfiniteLoading from 'vue-infinite-loading/src/components/InfiniteLoading.vue'
// Images
import bannerBackground from '~/assets/img/banner.png'
import bannerMembers from '~/assets/img/banner-members.png'
// Components
import MemberSearchResultCard from '~/components/MemberSearchResultCard'
import Spinner from '~/components/Spinner'
import MembersFilters from '~/components/MembersFilters'
// Queries
import MembersPrefetchQuery from '~/apollo/queries/MemberLandingPage/MembersPrefetch'
import MembersQuery from '~/apollo/queries/MemberLandingPage/Members'
import StateListQuery from '~/apollo/queries/StateList'

export default {
  head () {
    return {
      title: `${this.$t('membersPage.title')} | ${this.$t('site.title')}`,
      meta: [
        { hid: 'description', name: 'description', content: this.$t('membersPage.description') },
        { property: 'og:url', content: `${appConfig.site.url}/${this.locale}/bills` },
        {
          property: 'og:title',
          content: `${this.$t('membersPage.title')} | ${this.$t('site.title')}`
        },
        { property: 'og:description', content: this.$t('membersPage.description') }
      ]
    }
  },
  components: {
    MembersFilters,
    InfiniteLoading,
    MemberSearchResultCard,
    Spinner
  },
  data () {
    return {
      members: [],
      memberIds: [],
      page: 0,
      pageSize: 10,
      filterLoading: false,
      filterData: {},
      bannerBackground,
      bannerMembers,
      bannerStyle: `background-image: url("${bannerBackground}"); background-size: cover;`
    }
  },
  computed: {
    locale () {
      // when locale changes, reset the current page
      this.resetPage()
      return this.$store.state.locale
    },
    congress () {
      return [this.$store.state.currentCongress]
    },
    selectedStates () {
      return this.filterData.selectedStates ? this.filterData.selectedStates : []
    }
  },
  methods: {
    resetPage () {
      if (this.$refs.infiniteLoading) {
        this.$refs.infiniteLoading.stateChanger.reset()
      }
      this.members = []
      this.memberIds = []
      this.page = 0
    },
    prefetchMemberIds () {
      return this.$apollo.query({
        query: MembersPrefetchQuery,
        variables: { lang: this.locale, congress: this.congress, states: this.selectedStates }
      })
    },
    fetchMembers (ids) {
      return this.$apollo.query({
        query: MembersQuery,
        variables: { lang: this.locale, ids: ids }
      })
    },
    getCurrentPageItems () {
      return this.memberIds.filter(
        (id, index) => index >= this.page * this.pageSize && index < (this.page + 1) * this.pageSize
      )
    },
    async moreItems ($state) {
      // make sure memberIds is fetched
      if (!this.memberIds.length) {
        try {
          let result = await this.prefetchMemberIds()
          console.log('prefetch ids: ', result)
          this.memberIds = result.data.members[0].prefetchIds
        } catch (error) {
          console.log('no data :(', error)
        }
      }

      const items = this.getCurrentPageItems()

      if (items.length > 0) {
        this.fetchMembers(items)
          .then(({ data }) => {
            this.filterLoading = false
            const membersMap = _.keyBy(data.members, 'id')
            const members = items.map(id => membersMap[id])
            this.members = [...this.members, ...members]
            this.page++
            $state.loaded()
            console.log('BBBBB', data.members)
          })
          .catch(error => {
            this.filterLoading = false
            console.log('get members error', error)
            $state.complete()
          })
      } else {
        this.filterLoading = false
        $state.complete()
      }
    },
    filterMembers (filterData) {
      this.filterLoading = true
      this.resetPage()
      this.filterData = filterData
      console.log('filterData', filterData.selectedStates)
    }
  },
  apollo: {
    states: {
      query: StateListQuery,
      fetchPolicy: 'cache-and-network',
      variables () {
        return { lang: this.locale, stateList: true }
      },
      update (data) {
        return JSON.parse(data.maps[0].states)
      }
    }
  }
}
</script>

<style scoped lang="scss">
@import 'assets/css/app';
@import 'assets/css/colors';
@import 'assets/css/typography';

.banner {
  .banner-wrapper {
    height: 180px;
    display: flex;
    justify-content: space-between;
    @extend .pageWrapper-large;

    .text-container {
      order: 0;
      padding-top: 50px;

      .banner-title {
        display: inline-block;
        font-size: 2em;
        font-weight: $twMedium;
        line-height: 1.1em;
        text-align: left;
        letter-spacing: 0.02em;
        color: $twWhite;
        margin-top: 15px;
        margin-right: 20px;
      }
    }

    .image-container {
      .front-img {
        width: 200px;
        margin: auto 30px 10px 0;
      }
    }
  }
}

.members-section {
  padding: 40px 0;

  .members-section-wrapper {
    @extend .pageWrapper-large;
  }
}

.filters {
  padding-right: 40px;
  margin-bottom: 20px;
}

// desktop
@media screen and (min-width: $mediumDeviceWidth) {
  .image-container {
    order: 1;
    display: flex;
    margin-right: 100px;
  }
}

// tablet
@media screen and (max-width: $mediumDeviceWidth - 1) {
  .banner-wrapper {
    .image-container {
      display: none;
    }
  }
}

// phone
@media screen and (max-width: $smallDeviceWidth - 1) {
  .banner-wrapper {
    .image-container {
      display: none;
    }
  }

  .filters {
    padding-right: 0px;
  }
}
</style>

<style lang="scss">
.ivu-row {
  position: inherit;
}
</style>
