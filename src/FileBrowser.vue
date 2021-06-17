<template>
    <v-card class="mx-auto" :loading="loading > 0">
        <toolbar
            :path="path"
            :storages="storagesArray"
            :storage="activeStorage"
            :endpoints="endpoints"
            :axios="axiosInstance"
            v-on:storage-changed="storageChanged"
            v-on:path-changed="pathChanged"
            v-on:add-files="addUploadingFiles"
            v-on:folder-created="refreshPending = true"
            
        ></toolbar>
        <v-row no-gutters>
            <v-col v-if="tree && $vuetify.breakpoint.smAndUp" sm="auto">
                <tree
                    :path="path"
                    :storage="activeStorage"
                    :icons="icons"
                    :endpoints="endpoints"
                    :axios="axiosInstance"
                    :refreshPending="refreshPending"
                    v-on:path-changed="pathChanged"
                    v-on:loading="loadingChanged"
                    v-on:refreshed="refreshPending = false"
                    
                ></tree>
            </v-col>
            <v-divider v-if="tree" vertical></v-divider>
            <v-col>
                <list
                    :path="path"
                    :storage="activeStorage"
                    :icons="icons"
                    :endpoints="endpoints"
                    :axios="axiosInstance"
                    :refreshPending="refreshPending"
                    v-on:path-changed="pathChanged"
                    v-on:loading="loadingChanged"
                    v-on:refreshed="refreshPending = false"
                    v-on:file-deleted="refreshPending = true"
                    v-on:text-message="Text_Message"
                ></list>
            </v-col>
        </v-row>
        <upload
            v-if="uploadingFiles !== false"
            :path="path"
            :storage="activeStorage"
            :storages="storagesArray"
            :files="uploadingFiles"
            :icons="icons"
            :axios="axiosInstance"
            :endpoint="endpoints.upload"
            :maxUploadFilesCount="maxUploadFilesCount"
            :maxUploadFileSize="maxUploadFileSize"
            v-on:add-files="addUploadingFiles"
            v-on:remove-file="removeUploadingFile"
            v-on:clear-files="uploadingFiles = []"
            v-on:cancel="uploadingFiles = false"
            v-on:uploaded="uploaded"
            v-on:text-message="Text_Message"
        ></upload>
        <div>
            <v-snackbar
                v-model="snackbar"
                :timeout="3000"
                fixed
                top
                :color="options.color"
                class="text-left"
                >
                <span class="body-2">{{ options.messageText }}</span>
                <template v-slot:action="{ attrs }">
                    <v-icon v-bind="attrs" right color="white"
                    >{{ options.color == "green" ? "mdi-check" : "mdi-alert-circle" }}
                    </v-icon>
                </template>
            </v-snackbar>
        </div>
         
    </v-card>
</template>

<script>
import axios from "axios";

import Toolbar from "./Toolbar.vue";
import Tree from "./Tree.vue";
import List from "./List.vue";
import Upload from "./Upload.vue";

const availableStorages = [
    {
        name: "Local",
        code: "local",
        icon: "mdi-folder-multiple-outline"
    },
    {
        name: "Amazon S3",
        code: "s3",
        icon: "mdi-amazon-drive"
    }
    /*{
        name: "Dropbox",
        code: "dropbox",
        icon: "mdi-dropbox"
    }*/
];

const endpoints = {
    list: { url: "/localstorage/{storage}/list?path={path}", method: "get" },
    download: { url: "/localstorage/{storage}/download?path={path}", method: "get" },
    upload: { url: "/localstorage/{storage}/upload?path={path}", method: "post" },
    mkdir: { url: "/localstorage/{storage}/mkdir?path={path}", method: "post" },
    delete: { url: "/localstorage/{storage}/delete?path={path}", method: "post" }
};

const fileIcons = {
    zip: "mdi-folder-zip-outline",
    rar: "mdi-folder-zip-outline",
    htm: "mdi-language-html5",
    html: "mdi-language-html5",
    js: "mdi-nodejs",
    json: "mdi-json",
    py:"mdi-language-python",
    hex:"mdi-file-code-outline",
    go:"mdi-language-go",
    sh:"M 12.531 -0.198 L 4.662 -0.198 C 3.575 -0.198 2.695 0.697 2.695 1.802 L 2.695 17.802 C 2.695 18.907 3.575 19.802 4.662 19.802 L 16.465 19.802 C 17.552 19.802 18.432 18.907 18.432 17.802 L 18.432 5.802 L 12.531 -0.198 M 14.775 17.774 L 4.9719999999999995 17.774 L 4.9719999999999995 3.774 L 9.857 3.774 L 9.857 8.774000000000001 L 14.775 8.774000000000001 L 14.775 17.774 Z M 10.328 16.877 C 10.113 17.015 9.813 17.084 9.429 17.084 C 9.027 17.084 8.741 17.014 8.569 16.872 C 8.397 16.73 8.311 16.528 8.311 16.264 L 8.311 16.148 L 9.019 16.148 L 9.019 16.231 C 9.019 16.298000000000002 9.027 16.358 9.041 16.413 C 9.053 16.466 9.079 16.513 9.115 16.551 C 9.152 16.588 9.199 16.618 9.261 16.639 C 9.321 16.661 9.399000000000001 16.671 9.491 16.671 C 9.598 16.671 9.696 16.643 9.781 16.584 C 9.869 16.528 9.911 16.439 9.911 16.319 C 9.911 16.254 9.903 16.199 9.881 16.152 C 9.861 16.106 9.825 16.065 9.774000000000001 16.027 C 9.725 15.988 9.66 15.956 9.58 15.925 C 9.499 15.894 9.401 15.864 9.282 15.832 C 9.126 15.788 8.989 15.741 8.874 15.69 C 8.759 15.64 8.663 15.581 8.586 15.513 C 8.508 15.444 8.452 15.366 8.415 15.276 C 8.378 15.186 8.362 15.083 8.362 14.965 C 8.362 14.683 8.458 14.473 8.652 14.334 C 8.846 14.196 9.113 14.126 9.451 14.126 C 9.609 14.126 9.754 14.139 9.886 14.167 C 10.019 14.196 10.136 14.241 10.231 14.303 C 10.328 14.365 10.404 14.447 10.457 14.543 C 10.512 14.641 10.539 14.757 10.539 14.893 L 10.539 14.973 L 9.861 14.973 C 9.861 14.837 9.831 14.732 9.771 14.659 C 9.712 14.586 9.614 14.549 9.475 14.549 C 9.397 14.549 9.331 14.558 9.28 14.577 C 9.228 14.596 9.186 14.62 9.154 14.65 C 9.123 14.681 9.099 14.717 9.087 14.758 C 9.078 14.797 9.071 14.839 9.071 14.881 C 9.071 14.97 9.094 15.044 9.14 15.104 C 9.187 15.165 9.285 15.221 9.435 15.271 L 9.981 15.462 C 10.116 15.511 10.227 15.56 10.312 15.613 C 10.397 15.664 10.466 15.721 10.516 15.781 C 10.568 15.841 10.602 15.906 10.622 15.98 C 10.641 16.051 10.652 16.132 10.652 16.218 C 10.652 16.52 10.544 16.74 10.328 16.877 Z M 13.975 17.15 L 13.162 17.15 L 13.162 15.836 L 12.216 15.836 L 12.216 17.15 L 11.402 17.15 L 11.402 14.19 L 12.216 14.19 L 12.216 15.321 L 13.162 15.321 L 13.162 14.19 L 13.975 14.19 Z",
    php:"mdi-language-php",
    jar:"mdi-language-java",
    ts:"mdi-language-typescript",
    md: "mdi-markdown",
    apk:"android",
    pdf: "mdi-file-pdf",
    png: "mdi-file-image",
    jpg: "mdi-file-image",
    jpeg: "mdi-file-image",
    mp4: "mdi-filmstrip",
    mkv: "mdi-filmstrip",
    avi: "mdi-filmstrip",
    wmv: "mdi-filmstrip",
    mov: "mdi-filmstrip",
    txt: "mdi-file-document-outline",
    xls: "mdi-file-excel",
    pptx:"mdi-microsoft-powerpoint",
    pptx:"mdi-microsoft-powerpoint",
    accdb:"mdi-microsoft-access",
    docx:"mdi-microsoft-word",
    doc:"mdi-microsoft-word",
    other: "mdi-file-outline"
};

export default {
    components: {
        Toolbar,
        Tree,
        List,
        Upload
    },
    model: {
        prop: "path",
        event: "change"
    },
    props: {
        // comma-separated list of active storage codes
        storages: {
            type: String,
            default: () => availableStorages.map(item => item.code).join(",")
        },
        // code of default storage
        storage: { type: String, default: "local" },
        // show tree view
        tree: { type: Boolean, default: true },
        // file icons set
        icons: { type: Object, default: () => fileIcons },
        // custom backend endpoints
        endpoints: { type: Object, default: () => endpoints },
        // custom axios instance
        axios: { type: Function },
        // custom configuration for internal axios instance
        axiosConfig: { type: Object, default: () => {} },
        // max files count to upload at once. Unlimited by default
        maxUploadFilesCount: { type: Number, default: 0 },
        // max file size to upload. Unlimited by default
        maxUploadFileSize: { type: Number, default: 0 }
    },
    data() {
        return {
            loading: 0,
            path: "",
            activeStorage: null,
            uploadingFiles: false, // or an Array of files
            refreshPending: false,
            axiosInstance: null,
            snackbar: false,
            //messageText: null,
            message_text: {
                state:false,
                message:"",
                type:"success"
            },
            options: {
                color: "green",
                width: 300,
                position:"right",
                messageText:""
            }
        };
    },
    computed: {
        storagesArray() {
            //this.dir=storages
            let storageCodes = this.storages.split(","),
                result = [];
            storageCodes.forEach(code => {
                result.push(availableStorages.find(item => item.code == code));
            });
            return result;
        }
    },
    methods: {
        getMessage(open=false, message = "", type = "success"){
            console.log("Message",open,message,type);
            this.snackbar = open;
            this.options.color =
            type == "error"
                ? "red"
                : type == "success"
                ? "green"
                : type == "warning"
                ? "orange"
                : "cyan";
            this.options.messageText = message;
            return this.options
        },
        Text_Message(state=false, message = "", type = "success"){
            this.message_text.state=state;
            this.message_text.message=message;
            this.message_text.type=type
            this.$emit("test-message",this.message_text)
        },
        loadingChanged(loading) {
            if (loading) {
                this.loading++;
            } else if (this.loading > 0) {
                this.loading--;
            }
        },
        storageChanged(storage) {
            this.activeStorage = storage;
        },
        addUploadingFiles(files) {
            files = Array.from(files);

            if (this.maxUploadFileSize) {
                files = files.filter(
                    file => file.size <= this.maxUploadFileSize
                );
            }

            if (this.uploadingFiles === false) {
                this.uploadingFiles = [];
            }
            
            if (this.maxUploadFilesCount && this.uploadingFiles.length + files.length > this.maxUploadFilesCount) {
                files = files.slice(0, this.maxUploadFilesCount - this.uploadingFiles.length);
            }

            this.uploadingFiles.push(...files);
        },
        removeUploadingFile(index) {
            this.uploadingFiles.splice(index, 1);
        },
        uploaded() {
            this.uploadingFiles = false;
            this.refreshPending = true;
        },
        pathChanged(path) {
            this.path = path;
            this.$emit("change", path);
        }
    },
    created() {
        this.activeStorage = this.storage;
        this.axiosInstance = this.axios || axios.create(this.axiosConfig);
    },
    mounted() {
        if (!this.path && !(this.tree && this.$vuetify.breakpoint.smAndUp)) {
            this.pathChanged("/");
        }
    }
};
</script>

<style lang="scss" scoped>
</style>