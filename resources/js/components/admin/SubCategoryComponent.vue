<template>
  <div>
    <v-skeleton-loader
      class="mx-auto"
      type="table-heading,table-thead,table-tbody"
      v-show="skeleton"
    ></v-skeleton-loader>
    <v-data-table
      :headers="headers"
      :items="data"
      :items-per-page="data.length"
      class="elevation-1"
      :search="search"
      hide-default-footer
      v-show="!skeleton"
    >
      <template v-slot:item.directories="{ item }">
        <span>{{implodeDirectoryNames(item.directories)}}</span>
      </template>
      <template v-slot:top>
        <v-toolbar flat color="white">
          <v-toolbar-title>Подкатегории</v-toolbar-title>
          <v-divider class="mx-4" inset vertical></v-divider>
          <v-dialog v-model="dialog" max-width="450px">
            <template v-slot:activator="{ on }">
              <v-row>
                <v-col cols="8">
                  <v-btn color="success" dark class="mb-2" v-on="on">Добавить</v-btn>
                </v-col>
                <v-col cols="4">
                  <v-text-field
                    v-model="search"
                    append-icon="mdi-magnify"
                    label="Поиск"
                    single-line
                    hide-details
                    class="searchInput"
                  ></v-text-field>
                </v-col>
              </v-row>
            </template>
            <v-card>
              <v-card-title>
                <span class="headline">{{ formTitle }}</span>
              </v-card-title>

              <v-card-text>
                <v-container>
                  <v-row>
                    <v-col cols="12">
                      <v-select
                        :items="categories"
                        item-value="id"
                        item-text="name"
                        v-model="editedItem.category_id"
                        label="Категория"
                      ></v-select>
                    </v-col>
                  </v-row>
                  <v-row>
                    <v-col cols="12">
                      <v-text-field v-model="editedItem.name" label="Название"></v-text-field>
                    </v-col>
                  </v-row>
                  <v-row>
                    <v-col cols="12">
                      <v-combobox
                        v-model="editedItem.directories"
                        :items="directoryTypes"
                        item-value="id"
                        item-text="name"
                        label="Тех характеристики"
                        multiple
                        chips
                      ></v-combobox>
                    </v-col>
                  </v-row>
                </v-container>
              </v-card-text>

              <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn color="error darken-1" text @click="close">Отмена</v-btn>
                <v-btn color="success darken-1" text @click="save">Сохранить</v-btn>
              </v-card-actions>
            </v-card>
          </v-dialog>
        </v-toolbar>
      </template>
      <template v-slot:item.actions="{ item }">
        <v-icon color="warning" small class="mr-2" @click="editItem(item)">mdi-pencil</v-icon>
        <v-icon color="error" small @click="showDeleteDialog(item)">mdi-delete</v-icon>
      </template>
      <template v-slot:no-data>Нет данных !</template>
    </v-data-table>
    <v-row>
      <v-dialog justify="center" v-model="deleteDialog" persistent max-width="350">
        <v-card>
          <v-card-title
            class="headline deleteDialogHead"
          >Вы действительно хотите удалить выбранную строку ?</v-card-title>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="error darken-1" text @click="deleteDialog = false">Нет</v-btn>
            <v-btn color="success darken-1" text @click="deleteItem">Да</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-row>
    <v-snackbar v-model="snackbar.show" right top :color="snackbar.color" :timeout="snackbar.time">
      {{ snackbar.text }}
      <template v-slot:action="{ attrs }">
        <v-icon @click="snackbar.show = false" color="white">mdi-close-circle-outline</v-icon>
      </template>
    </v-snackbar>
  </div>
</template>

<script>
import main from "../../mixins/main.js";
export default {
  mixins: [main],
  data: () => ({
    directoryTypes: [],
    categories: [],
    headers: [
      { text: "Категория", value: "category.name", sortable: false },
      { text: "Название", value: "name", sortable: false },
      { text: "Справочники", value: "directories", sortable: false },
      {
        text: "category_id",
        value: "category_id",
        sortable: false,
        align: " d-none",
      },
      { text: "Действия", value: "actions", sortable: false },
    ],
    editedItem: {
      name: "",
      category_id: "",
      directories: [],
    },
    defaultItem: {
      name: "",
      category_id: "",
      directories: [],
    },
  }),
  created() {
    this.index();
  },
  methods: {
    index() {
      axios
        .get("/api/subCategories")
        .then((response) => {
          var res = response.data;
          if (res.success) {
            this.data = res.data.subCategories;
            this.skeleton = false;
            this.categories = res.data.categories;
            this.directoryTypes = res.data.directoryTypes;
          } else {
            alert(res.msg);
          }
        })
        .catch(function (error) {});
    },
    store() {
      axios
        .post("/api/subCategories", this.editedItem)
        .then((response) => {
          var res = response.data;
          if (res.success) {
            this.showSnack("success", "Данные успешно добавлены !");
            this.index();
            this.close();
          } else {
            alert(res.msg);
          }
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    update() {
      axios
        .put("/api/subCategories/" + this.editedItem.id, this.editedItem)
        .then((response) => {
          var res = response.data;
          if (res.success) {
            this.showSnack("success", "Данные успешно изменены !");
            this.editedItem["category.name"] = this.getName(
              this.editedItem.category_id,
              this.categories
            );
            Object.assign(this.data[this.editedIndex], this.editedItem);
            this.close();
          } else {
            alert(res.msg);
          }
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    deleteItem() {
      axios
        .delete("/api/subCategories/" + this.editedItem.id)
        .then((response) => {
          var res = response.data;
          if (res.success) {
            this.showSnack("success", "Данные успешно удалены !");
            this.data.splice(this.editedIndex, 1);
            this.close();
          } else {
            alert(res.msg);
          }
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    implodeDirectoryNames(directory) {
      var namesArr = [];
      var namesStr = "";
      for (var key in directory) {
        namesArr.push(directory[key].name);
      }
      namesStr = namesArr.join(", ");
      return namesStr;
    },
  },
};
</script>