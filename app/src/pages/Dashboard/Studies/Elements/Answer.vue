<template>
	<div class="sc-answer" v-loading="loading">
		<activity-form
			v-if="! answer.content.raw"
			ref="answerForm"
			elClass="sc-activity--answer"
			component="groups"
			type="answer_update"
			v-on:activitySaved="addAnswer"
			placeholder="Enter your answer..."
			:activityID="this.answer.id"
			:primaryItem="this.groupData.id"
			:secondaryItem="this.questionData.id"></activity-form>

		<activity v-if="answer.content.raw" :activity="answer" :showContent="true"></activity>

		<p class="category">
			<span v-html="groupAnswerText"></span>&nbsp;
			<a href="#" @click.prevent="showAnswers = false" v-if="hasGroupAnswers && showAnswers">(hide)</a>
			<a href="#" @click.prevent="showAnswers = true" v-if="!showAnswers && hasGroupAnswers">(show)</a>
		</p>

		<div v-if="showGroupAnswers" class="sc-answer--group">
			<activity v-for="gAnswer in groupAnswers" class="sc-question--group-answers--answer" :activity="gAnswer" :showContent="true" :showForm="true"></activity>
		</div>
	</div>
</template>
<script>
  import { Activity, ActivityForm } from 'src/components';
  import { mapState, mapGetters } from 'vuex';

  function getDefaultData () {
    return {
      showAnswers : true,
      update      : false,
      loading     : true,
      answer      : {
        date   : 0,
        content: {
          raw: ''
        }
      },
      groupAnswers: [],
    };
  }

  export default {
    name: 'sc-answer',

    components: {
      ActivityForm,
      Activity
    },
    data      : getDefaultData,
    props     : {
      questionData: Object,
    },
    computed  : {
      ...mapState(['user', 'group']),
      ...mapGetters('group', ['isOrgAdmin', 'isGroupAdmin']),

      groupData() {
        return this.group.group;
      },

      hasGroupAnswers() {
        if (!this.groupData.id) {
          return false;
        }

        return this.groupAnswers.length;
      },

      showGroupAnswers() {
        return this.hasGroupAnswers && this.showAnswers && (
            Boolean(this.answer.content.raw) || this.isGroupAdmin
          );
      },

      groupAnswerText() {
        if (!this.groupData.id) {
          return '';
        }

        if (this.questionData.is_private) {
          return 'This question is private, your answer will not be shared with the group.';
        }

        if (!this.groupAnswers.length) {
          return 'No Group Answers Yet';
        } else if (1 === this.groupAnswers.length) {
          return this.groupAnswers.length + ' Group Answer';
        } else {
          return this.groupAnswers.length + ' Group Answers';
        }
      },
    },
    watch     : {
      '$route' () {
        this.reset();
        this.getGroupAnswers();
      }
    },

    methods: {
      addAnswer(answer) {
        this.answer = answer;
      },
      addComment(comment) {

      },
      handleEditAnswer(event) {
        event.preventDefault();
        this.update = true;
        this.$nextTick(() => {
          this.$refs.answerForm.updateComment(this.answer.content.raw);
          this.$refs.answerForm.setFocus();
        })
      },
      cancelEdit(e) {
        this.update = false;
      },
      getGroupAnswers() {

        let params = 'context=edit&_embed=true&component=groups&show_hidden=true&display_comments=threaded&per_page=20&secondary_id=' + this.questionData.id + '&primary_id=' + this.groupData.id;

        if (this.questionData.is_private) {
          params += '&user=' + this.$store.state.user.me.id;
        }

        this.$http
          .get(
            '/wp-json/studychurch/v1/activity/?' + params)
          .then(response => {
            this.loading = false;
            if (response.data.length) {
              this.groupAnswers = [];
              for (let i = 0; i < response.data.length; i++) {
                if (this.$store.state.user.me.id === response.data[i].user) {
                  this.answer = response.data[i];
                } else {
                  this.groupAnswers.push(response.data[i]);
                }
              }
            }
          })
          .finally(() => {
            this.loading = false;
            this.$root.reftag();
          })
      },
      reset (keep) {
        let def = getDefaultData();
        def[keep] = this[keep];
        Object.assign(this.$data, def);
      }
    },
    mounted() {
      this.getGroupAnswers();
    },
  };
</script>
