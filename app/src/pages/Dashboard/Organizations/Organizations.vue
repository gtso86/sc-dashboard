<template>
	<div class="sc-group">

		<div class="row">
			<div class="col-lg-4">
				<card class="card-user">
					<div slot="image" class="image">
						<img src="@/assets/img/bg-bible.jpg" alt="...">
					</div>
					<div>
						<div class="author">
							<img class="avatar border-gray" :src="groupData.avatar_urls.full" alt="..." v-if="hasAvatar" style="background: white;">
							<font-awesome-icon icon="church" class="avatar border-gray" v-else></font-awesome-icon>
							<router-link :to="'/groups/' + $route.params.slug + '/'">
								<h5 class="title" v-html="groupData.name"></h5></router-link>
							<p class="description" v-show="showGroupDesc" v-html="groupData.description.rendered"></p>
							<p class="description" v-show="showGroupDesc">
								<a href="#" @click.stop="showGroupDesc=false">Hide details</a></p>
							<p class="description" v-show="!showGroupDesc">
								<a href="#" @click.stop="showGroupDesc=true">Show details</a></p>
						</div>
					</div>
				</card>

			</div>

			<div class="col-lg-8" v-loading="!group.organization.id" style="min-height: 500px;">

				<el-menu :default-active="defaultActiveTab" class="el-menu-demo" mode="horizontal" :router="true">
					<el-menu-item :index="'/organizations/' + this.$route.params.slug + '/'"><font-awesome-icon icon="comments"></font-awesome-icon>&nbsp;&nbsp;<span>Discussion</span></el-menu-item>
					<el-menu-item :index="'/organizations/' + this.$route.params.slug + '/groups/'"><font-awesome-icon icon="users"></font-awesome-icon>&nbsp;&nbsp;<span>Groups</span></el-menu-item>
					<el-menu-item :index="'/organizations/' + this.$route.params.slug + '/studies/'"><font-awesome-icon icon="book"></font-awesome-icon>&nbsp;&nbsp;<span>Library</span></el-menu-item>
					<el-menu-item :index="'/organizations/' + this.$route.params.slug + '/members/'"><font-awesome-icon icon="user"></font-awesome-icon>&nbsp;&nbsp;<span>Members</span></el-menu-item>
					<el-menu-item :index="'/organizations/' + this.$route.params.slug + '/settings/'" v-if="isOrgAdmin()"><font-awesome-icon icon="cogs"></font-awesome-icon>&nbsp;&nbsp;<span>Settings</span></el-menu-item>
				</el-menu>

				<br />

				<router-view :groupData.sync="group.organization" v-if="group.organization.id"></router-view>

			</div>
		</div>

	</div>
</template>
<script>
  import {
    Card,
    Table as NTable
  } from 'src/components'

  import { Menu, MenuItem } from 'element-ui';
  import { mapState, mapGetters } from 'vuex';

  function getDefaultData () {
    return {
      loading : false,
      showGroupDesc : true,
    }
  }

  export default {
    components: {
      'el-menu'     : Menu,
      'el-menu-item': MenuItem,
      Card,
      NTable
    },
    data      : getDefaultData,
    created() {
      this.setupCurrentGroup();
    },
    watch     : {
      '$route' (to, from) {
        if (to.params.slug === from.params.slug) {
          return;
        }

        this.setupCurrentGroup();
      }
    },
    computed  : {
      ...mapState(['user', 'group']),
      ...mapGetters('group', ['isOrgAdmin', 'isGroupAdmin']),
	  hasAvatar() {
        return -1 === this.groupData.avatar_urls.full.indexOf('mystery-group');
	  },
	  groupData() {
        return this.group.organization;
	  },
      defaultActiveTab() {
        return this.$route.path;
      }
    },
    methods   : {
      setupCurrentGroup () {
        this.$store
          .dispatch('group/fetchOrg', {id: this.$route.params.slug, key: 'slug'})
          .then();
      },
      /**
       * Fetch
       */
      reset (keep) {
        let def = getDefaultData();
        def[keep] = this[keep];
        Object.assign(this.$data, def);
      }
    }
  }
</script>
<style>
</style>
