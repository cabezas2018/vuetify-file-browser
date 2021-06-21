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
        name: "Media SDK",
        code: "media",
        icon: "mdi-folder-star-multiple-outline"
    }
    /*{
        name: "Amazon S3",
        code: "s3",
        icon: "mdi-amazon-drive"
    }*/
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
    gz: "mdi-folder-zip-outline",
    htm: "mdi-language-html5",
    html: "mdi-language-html5",
    js: "mdi-nodejs",
    json: "mdi-json",
    py:"mdi-language-python",
    hex:"mdi-file-code-outline",
    go:"mdi-language-go",
    sh:"M 12.087 18.491 C 11.578 18.812 10.869 18.973 9.961 18.973 C 9.013 18.973 8.336 18.808 7.928 18.48 C 7.522 18.149 7.317 17.678 7.317 17.062 L 7.317 16.791 L 8.994 16.791 L 8.994 16.986 C 8.994 17.141 9.013 17.283 9.044 17.408 C 9.075 17.535 9.135 17.643 9.219 17.73 C 9.305 17.819 9.42 17.887 9.562 17.936 C 9.707 17.985 9.887 18.011 10.105 18.011 C 10.361 18.011 10.592 17.944 10.795 17.81 C 11.001 17.678 11.104 17.469 11.104 17.191 C 11.104 17.042 11.078 16.913 11.028 16.804 C 10.98 16.695 10.896 16.597 10.779 16.511 C 10.661999999999999 16.424 10.509 16.346 10.318 16.273 C 10.128 16.202 9.893 16.129 9.615 16.054 C 9.242 15.955 8.92 15.845 8.65 15.727 C 8.375 15.609 8.15 15.471 7.966 15.312 C 7.784 15.153 7.649 14.97 7.564 14.762 C 7.479 14.552 7.438 14.312 7.438 14.037 C 7.438 13.379 7.667 12.889 8.124 12.566 C 8.582 12.242 9.213 12.081 10.014 12.081 C 10.385 12.081 10.729 12.113 11.042 12.179 C 11.357 12.244 11.629 12.348 11.858 12.496 C 12.087 12.642 12.267 12.828 12.393 13.053 C 12.52 13.281 12.587 13.552 12.587 13.869 L 12.587 14.056 L 10.98 14.056 C 10.98 13.739 10.909 13.495 10.77 13.322 C 10.631 13.153 10.398 13.067 10.071 13.067 C 9.885 13.067 9.729 13.09 9.608 13.131 C 9.484 13.175 9.384 13.232 9.311 13.304 C 9.238 13.376 9.182 13.461 9.156 13.554 C 9.128 13.647 9.115 13.745 9.115 13.843 C 9.115 14.051 9.167 14.221 9.278 14.361 C 9.387 14.502 9.62 14.63 9.974 14.747 L 11.266 15.195 C 11.585 15.307 11.845 15.424 12.047 15.547 C 12.248 15.668 12.41 15.798 12.53 15.939 C 12.65 16.078 12.732 16.23 12.78 16.399 C 12.827 16.565 12.85 16.755 12.85 16.957 C 12.85 17.66 12.596 18.172 12.087 18.491 Z M 17.477 18.772 L 16.056 18.772 L 16.056 15.81 L 14.398 15.81 L 14.398 18.772 L 12.975999999999999 18.772 L 12.975999999999999 12.097 L 14.398 12.097 L 14.398 14.649000000000001 L 16.056 14.649000000000001 L 16.056 12.097 L 17.477 12.097 Z M 14.434 2.051 L 6.434 2.051 C 5.324 2.051 4.434 2.951 4.434 4.051 L 4.434 20.051 C 4.434 21.161 5.324 22.051 6.434 22.051 L 18.434 22.051 C 19.544 22.051 20.434 21.161 20.434 20.051 L 20.434 8.051 L 14.434 2.051 M 18.384 20.051 L 6.384 20.051 L 6.384 4.051 L 13.384 4.051 L 13.384 9.051 L 18.384 9.051 L 18.384 20.051",
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
    ppt:"mdi-microsoft-powerpoint",
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