<template>

	<div class="sc-group--assignments" v-loading="loadingTodos" style="min-height: 200px;">

		<div class="text-right" v-if="isGroupAdmin()">
			<n-button type="primary" @click.native="clearStudyValues();showModal = true">Create To-Do</n-button>
		</div>
		<modal :show.sync="showModal" headerclasses="justify-content-center" v-loading="creatingTodo">
			<h4 slot="header" class="title title-up">Create a new To-Do</h4>

			<div v-for="study in newTodo.studies">
				<label :for="'study-' + study.id" v-html="study.title.rendered"></label>
				<p>
					<el-select v-model="study.value" :id="'study-' + study.id" multiple placeholder="Select" class="modal-select select-primary">
						<el-option
							class="select-primary"
							v-for="chapter in study.navigation"
							:key="chapter.id"
							:label="chapter.title.rendered"
							:value="chapter.id">
						</el-option>
					</el-select>
				</p>
			</div>

			<p>
				<label for="instructions">Instructions</label>
				<el-input
					ref="description"
					type="textarea"
					id="instructions"
					:autosize="{ minRows: 4 }"
					resize="none"
					label="Study Description"
					v-model="newTodo.description"></el-input>
			</p>

			<p>
				<label for="datepicker">Due Date</label>
				<fg-input>
					<el-date-picker id="datepicker" value-format="yyyy-MM-dd" v-model="newTodo.date" type="date" placeholder="Pick a day">
					</el-date-picker>
				</fg-input>
			</p>

			<template slot="footer">
				<n-button type="primary" @click.native="createTodo">Create</n-button>
			</template>
		</modal>

		<modal :show.sync="showEditModal" headerclasses="justify-content-center" v-loading="editingTodo">
			<h4 slot="header" class="title title-up">Edit To-Do</h4>

			<div v-for="study in editTodoData.studies">
				<label :for="'study-' + study.id" v-html="study.title.rendered"></label>
				<p>
					<el-select v-model="study.value" :id="'study-' + study.id" multiple placeholder="Select" class="modal-select select-primary">
						<el-option
								class="select-primary"
								v-for="chapter in study.navigation"
								:key="chapter.id"
								:label="chapter.title.rendered"
								:value="chapter.id">
						</el-option>
					</el-select>
				</p>
			</div>

			<p>
				<label for="instructions">Instructions</label>
				<el-input
						ref="description"
						type="textarea"
						id="instructions"
						:autosize="{ minRows: 4 }"
						resize="none"
						label="Study Description"
						v-model="editTodoData.content"></el-input>
			</p>

			<p>
				<label for="datepicker">Due Date</label>
				<fg-input>
					<el-date-picker id="datepicker" value-format="yyyy-MM-dd" v-model="editTodoData.formattedDate" type="date" placeholder="Pick a day">
					</el-date-picker>
				</fg-input>
			</p>

			<template slot="footer">
				<n-button type="primary" @click.native="saveEdit">Save Changes</n-button>
			</template>
		</modal>

		<card v-for="data in todoData" :class="'card todo'">
			&nbsp;
			<h6>Due Date: {{ data.date }}</h6>
			<p v-for="lesson in data.lessons">
				<router-link v-if="lesson.link != false" :to="'/groups/' + $route.params.slug + $root.cleanLink(lesson.link)">
					<i class="now-ui-icons design_bullet-list-67"></i>&nbsp;
					<span v-html="lesson.title"></span></router-link>
			</p>
			<p v-html="data.content"></p>
			<p class="todo-actions">
				<n-button type="info"
						  @click.native="editTodo( data )"
						  size="sm"
						  class="edit btn-neutral"
						  icon
						  v-if="isGroupAdmin()"><font-awesome-icon icon="edit"></font-awesome-icon>
				</n-button>
				<n-button type="danger"
						  @click.native="removeTodo( data.key )"
						  size="sm"
						  class="remove btn-neutral"
						  icon
						  v-if="isGroupAdmin()"><font-awesome-icon icon="times"></font-awesome-icon>
				</n-button>
			</p>
		</card>

		<p v-if="!todoData.length && !loadingTodos" class="text-center">There are no upcoming to-dos.</p>

	</div>

</template>
<script>
  import { Input, Message, Select, Option, DatePicker } from 'element-ui';
  import { mapState, mapGetters } from 'vuex';

  import {
    Card,
    Table as NTable,
    Button,
    Modal,
  } from 'src/components'

  function getDefaultData () {
    return {
      creatingTodo: false,
      showModal   : false,
		showEditModal: false,
		editingTodo: false,
      loadingTodos: true,
      loadingMore : false,
      todoData    : [],
      todoPage    : 1,
      newTodo     : {
        description: '',
        studies    : [],
        date       : ''
      },
		editTodoData: {}
    }
  }

  export default {
    components: {
      Card,
      NTable,
      Button,
      Modal,
      'el-select'     : Select,
      'el-option'     : Option,
      'el-date-picker': DatePicker

    },
    props     : {
      groupData: {
        default: {
          id     : 0,
          studies: []
        }
      },
    },
    data      : getDefaultData,
    mounted() {
      this.getGroupTodos();
      this.getStudies();
    },
    computed  : {
      ...mapState(['user', 'group']),
      ...mapGetters('user', ['getUserById']),
      ...mapGetters('group', ['isGroupAdmin', 'isGroupAdmin', 'getGroupMembers', 'getGroupAdmins']),

      leaders() {
        return this.groupData.members.filter(member => member.admin);
      },
      members() {
        return this.groupData.members.filter(member => !member.admin);
      }
    },
    methods   : {
      createTodo() {
        if (!this.newTodo.date || !this.newTodo.description) {
          Message.error('Please enter a date and description for your new to-do item');
          return;
        }

        this.creatingTodo = true;
        let studies = [];
        for (let i = 0; i < this.newTodo.studies.length; i++) {
          studies = studies.concat(this.newTodo.studies[i].value);
        }

        this.$http.post('/wp-json/studychurch/v1/assignments/', {
          group_id: this.groupData.id,
          content : this.newTodo.description,
          lessons : studies,
          date    : this.newTodo.date,
        })
          .then(response => {
            this.getGroupTodos();
            this.creatingTodo = false;
          })
      },
		editTodo( itemId ) {

          this.showEditModal = true;

          this.editTodoData = itemId;

          if ( undefined === this.editTodoData.studies ) {
            this.editTodoData.studies = [];
		  }

          this.editTodoData.content = this.stripHTML( this.editTodoData.content );

            let savedStudy = null;

            if ( this.editTodoData.lessons.length ) {
                for (let z = 0; z < this.editTodoData.lessons.length; z++) {

                    savedStudy = this.editTodoData.lessons.length > 0 ? this.editTodoData.lessons[z].id : false;

                    if (savedStudy) {
                        for (let i = 0; i < this.newTodo.studies.length; i++) {
                            let item = this.newTodo.studies[i].navigation;

                            this.newTodo.studies[i].value = [];

                            for (let y = 0; y < item.length; y++) {

                                if (item[y].id === savedStudy) {
                                    this.newTodo.studies[i].value.push(item[y].id);
                                }
                            }
                        }

                        this.editTodoData.studies = this.newTodo.studies;
                    }
                }
            } else {

                this.clearStudyValues();

                this.editTodoData.studies = this.newTodo.studies;
			}

            //

		},
		clearStudyValues() {
            for (let i = 0; i < this.newTodo.studies.length; i++) {
                this.newTodo.studies[i].value = [];
            }
		},
		saveEdit() {

          this.editingTodo = true;

            let studies = [];
            for (let i = 0; i < this.editTodoData.studies.length; i++) {
                studies = studies.concat(this.editTodoData.studies[i].value);
            }

            this.$http.post('/wp-json/studychurch/v1/assignments/edit', {
                id: this.editTodoData.key,
                content: this.editTodoData.content,
                lessons : studies,
				date: this.editTodoData.formattedDate,
            }).then(response => {
                if ( response.data.message.length ) {

                if ( response.data.success ) {
                    Message.success( response.data.message );
                } else {
                    Message.error( response.data.message );
                }


                this.getGroupTodos();
            } else {
                Message.error( 'An error occurred.' );
                this.loadingTodos = false;
            }

            this.editingTodo = false;
            this.showEditModal = false;
        });
		},
		removeTodo( itemId ) {

          this.loadingTodos = true;

          this.$http.delete('/wp-json/studychurch/v1/assignments/' + itemId, {
                assignment_id: itemId
			}).then(response => {
				if ( response.data.message.length ) {

				    if ( response.data.success ) {
                        Message.success( response.data.message );
					} else {
				        Message.error( response.data.message );
					}

					this.getGroupTodos();
				} else {
					Message.error( 'An error occurred.' );
					this.loadingTodos = false;
				}
			});
		},
      getStudies () {
        if (this.newTodo.studies.length) {
          return;
        }

        for (let i = 0; i < this.groupData.studies.length; i++) {
          this.getStudy(this.groupData.studies[i]);
        }
      },
      getStudy (study) {
        this.$http
          .get('/wp-json/studychurch/v1/studies/' + study.id + '/navigation')
          .then(response => {
            study.navigation = response.data;
            this.newTodo.studies.push(study);
          })
      },
      getGroupTodos () {
        this.loadingTodos = true;
        this.$http
          .get(
            '/wp-json/studychurch/v1/assignments?group_id=' + this.groupData.id)
          .then(response => {
              this.todoData = response.data;
              this.showModal = false;
            }
          )
          .finally(() => this.loadingTodos = false)
      },
        stripHTML( value ) {
            var div = document.createElement("div");
            div.innerHTML = value;
            var text = div.textContent || div.innerText || "";
            return text;
        },
      reset (keep) {
        let def = getDefaultData();
        def[keep] = this[keep];
        Object.assign(this.$data, def);
      }
    }
  }
</script>
<style>

	.todo-actions {
		position: absolute;
		right: 0;
		top: 0;
		display: block;
	}
</style>