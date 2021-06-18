<template>
    <v-card flat tile min-height="380" class="d-flex flex-column">
        <confirm ref="confirm"></confirm>
        
            <v-card-text
                v-if="!path"
                class="grow d-flex justify-center align-center grey--text"
            >Select a folder or a file</v-card-text>
            <v-card-text
                v-else-if="isFile"
                class="grow d-flex justify-center align-center"
            >File: {{ path }}</v-card-text>
            <v-card-text v-else-if="dirs.length || files.length" class="grow">
                <v-list subheader v-if="dirs.length">
                    <v-subheader>Folders</v-subheader>
                    <v-list-item
                        v-for="item in dirs"
                        :key="item.basename"
                        @click="changePath(item.path)"
                        class="pl-0"
                    >
                        <v-list-item-avatar class="ma-0">
                            <v-icon>mdi-folder-outline</v-icon>
                        </v-list-item-avatar>
                        <v-list-item-content class="py-2">
                            <v-list-item-title v-text="item.basename"></v-list-item-title>
                        </v-list-item-content>
                        <v-list-item-action>
                            <v-btn icon @click.stop="deleteItem(item)">
                                <v-icon color="red lighten-5">mdi-delete-outline</v-icon>
                            </v-btn>
                            <v-btn icon v-if="false">
                                <v-icon color="grey lighten-1">mdi-information</v-icon>
                            </v-btn>
                        </v-list-item-action>
                    </v-list-item>
                </v-list>
                <v-divider v-if="dirs.length && files.length"></v-divider>
                <v-list subheader v-if="files.length">
                    <v-subheader>{{$t("Files")}}
                        <v-chip small outlined>{{files.length}}</v-chip>
                    </v-subheader>
                    <v-list-item
                        v-for="item in files"
                        :key="item.basename"
                        @click="changePath(item.path)"
                        class="pl-0"
                    >
                        <v-list-item-avatar class="ma-0">
                            <v-icon>{{ icons[item.extension.toLowerCase()] || icons['other'] }}</v-icon>
                        </v-list-item-avatar>

                        <v-list-item-content class="py-2">
                            <v-list-item-title v-text="item.basename"></v-list-item-title>
                            <v-list-item-subtitle>{{ formatBytes(item.size) }}</v-list-item-subtitle>
                        </v-list-item-content>

                        <v-list-item-action>
                            <v-btn icon @click.stop="deleteItem(item)">
                                <v-icon color="red lighten-5">mdi-delete-outline</v-icon>
                            </v-btn>
                            <v-btn icon v-if="false">
                                <v-icon color="grey lighten-1">mdi-information</v-icon>
                            </v-btn>
                        </v-list-item-action>
                        <v-list-item-action>
                            <v-btn icon @click.stop="downloadItem(item)">
                                <v-icon color="primary lighten-5">mdi-cloud-download-outline</v-icon>
                            </v-btn>
                            <v-btn icon v-if="false">
                                <v-icon color="grey lighten-1">mdi-information</v-icon>
                            </v-btn>
                        </v-list-item-action>
                    </v-list-item>
                </v-list>
            </v-card-text>
            <v-card-text
                v-else-if="filter"
                class="grow d-flex justify-center align-center grey--text py-5"
            >No files or folders found</v-card-text>
            <v-card-text
                v-else
                class="grow d-flex justify-center align-center grey--text py-5"
            >The folder is empty</v-card-text>

       
        <v-divider v-if="path"></v-divider>
        <v-toolbar v-if="false && path && isFile" dense flat class="shrink">
            <v-btn icon >
                <v-icon>mdi-download</v-icon>
            </v-btn>
        </v-toolbar>
        <v-toolbar v-if="path && isDir" dense flat class="shrink">
            <v-text-field
                solo
                flat
                hide-details
                label="Filter"
                v-model="filter"
                prepend-inner-icon="mdi-filter-outline"
                class="ml-n3"
            ></v-text-field>
            <v-btn icon v-if="false">
                <v-icon>mdi-eye-settings-outline</v-icon>
            </v-btn>
            <v-btn   icon @click="load">
                <v-icon>mdi-refresh</v-icon>
            </v-btn>
        </v-toolbar>
    </v-card>
</template>

<script>
import { formatBytes } from "./util";
import Confirm from "./Confirm.vue";

export default {
    props: {
        icons: Object,
        storage: String,
        path: String,
        endpoints: Object,
        axios: Function,
        refreshPending: Boolean
    },
    components: {
        Confirm,
        
    },
    data() {
        return {
            items: [],
            filter: ""
        };
    },
    computed: {
        dirs() {
             if(!this.items){
                 console.warn('No files')
                 return false
             }
            return this.items.filter(
                item =>
                    item.type === "dir" && item.basename.includes(this.filter)
            );
        },
        files() {
             if(!this.items){
                 console.warn('No files')
                 return false
             }
            return this.items.filter(
                item =>
                    item.type === "file" && item.basename.includes(this.filter)
            );
        },
        isDir() {
            return this.path[this.path.length - 1] === "/";
        },
        isFile() {
            return !this.isDir;
        }
    },
    methods: {
        formatBytes,
        changePath(path) {
            this.$emit("path-changed", path);
        },
        async load() {
            this.$emit("loading", true);
            if (this.isDir) {
                let url = this.endpoints.list.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), this.path);

                let config = {
                    url,
                    method: this.endpoints.list.method || "get"
                };

                let response = await this.axios.request(config);
                this.items = response.data;
            } else {
                // TODO: load file
            }
            this.$emit("loading", false);
        },
        async deleteItem(item) {
            if(item.basename.indexOf("cloud__") > -1 || item.basename.indexOf("__cloud_") > -1 )return this.$emit("text-message", true,'Not file deleted','error');
            let confirmed = await this.$refs.confirm.open(
                "Delete",
                `Are you sure<br>you want to delete this ${
                    item.type === "dir" ? "folder" : "file"
                }?<br><em>${item.basename}</em>`
            );

            if (confirmed) {
                this.$emit("loading", true);
                let url = this.endpoints.delete.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), `'${item.path}'`);

                let config = {
                    url,
                    method: this.endpoints.delete.method || "post"
                };
                try {
                    await this.axios.request(config);
                    this.$emit("file-deleted");
                    this.$emit("loading", false);
                    this.$emit("text-message", true,'File deleted','success');
                   
                    
                } catch (error) {
                    console.warn(error);
                    this.$emit("loading", false);
                    this.$emit("text-message", true,'Not file deleted','error');
                    
                    
                }
            }
        },
        export(data, filename, ext = "xlsx") {
            var element = document.createElement("a");
            element.setAttribute("href", "data:text/plain;charset=utf-8," + encodeURIComponent(data));
            element.setAttribute("download", filename  + "." + ext);
            element.style.display = "none";
            document.body.appendChild(element);
            element.click();
            document.body.removeChild(element);
            
        },
        async downloadItem(item){
            if(item.basename.indexOf("cloud__") > -1 || item.basename.indexOf("__cloud_") > -1 )return this.$emit("text-message", true,'Not Donwload','error');
            //var item_file=item.basename.split('.')
            //console.log('down',item_file[0],item_file[1])
            let url = this.endpoints.download.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), `'${item.path}'`);

                let config = {
                    url,
                    method: this.endpoints.download.method || "get"
                };
                try {
                    var content= await this.axios.request(config);
                    this.export(content.data,item.name, item.extension);
                } catch (error) {
                    console.warn(error);
                    this.$emit("loading", false);
                }
        }
    },
    watch: {
        async path() {
            this.items = [];
            await this.load();
        },
        async refreshPending() {
            if (this.refreshPending) {
                await this.load();
                this.$emit("refreshed");
            }
        }
    }
};
</script>

<style lang="scss" scoped>
.v-card {
    height: 100%;
}
.scroll-y {
        overflow-y: auto;
        height:500px
    }
</style>