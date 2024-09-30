<template>
  <v-app>
    <v-container>
      <v-app-bar app color="teal" dark elevation="3">
        <v-toolbar-title>Todo List</v-toolbar-title>
      </v-app-bar>

      <v-row justify="center" class="mt-10">
        <v-col cols="12" md="8">
          <v-card elevation="3">
            <v-card-title>
              <v-text-field
                v-model="newTask"
                label="New Task"
                outlined
                dense
                required
                @keyup.enter="addTask"
              ></v-text-field>
              <v-spacer></v-spacer>
              <v-btn @click="addTask" color="teal" class="ml-2">Add</v-btn>
            </v-card-title>
          </v-card>
        </v-col>
      </v-row>

      <v-row justify="center" class="mt-2">
        <v-col cols="12" md="8">
          <v-list dense elevation="3">
            <v-list-item
              v-for="(task, index) in tasks"
              :key="index"
              :class="{ 'task-completed': task.completed }"
            >
              <v-list-item-content>
                <v-checkbox
                  v-model="task.completed"
                  @change="toggleTaskCompletion(task)"
                  :label="task.text"
                ></v-checkbox>
              </v-list-item-content>
              <v-list-item-action>
                <v-btn @click="deleteTask(index)" color="red">
                  delete
                </v-btn>
              </v-list-item-action>
            </v-list-item>
          </v-list>
        </v-col>
      </v-row>
    </v-container>
  </v-app>
</template>

<script>
export default {
  data() {
    return {
      newTask: '',
      tasks: []
    };
  },
  methods: {
    addTask() {
      if (this.newTask.trim() !== '') {
        this.tasks.push({ text: this.newTask, completed: false });
        this.newTask = '';
      }
    },
    deleteTask(index) {
      this.tasks.splice(index, 1);
    },
    toggleTaskCompletion(task) {
      task.completed = !task.completed;
    }
  }
};
</script>

<style scoped>
.task-completed {
  text-decoration: line-through;
  color: gray;
}
</style>