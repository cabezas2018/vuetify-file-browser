<template>
    <v-card class="mx-auto" :loading="loading > 0">
        <toolbar
            :path="path"
            :storages="storagesArray"
            :storage="activeStorage"
            :endpoints="endpoints"
            :axios="axiosInstance"
            :IconsToolbar="IconsToolbar"
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
                    v-on:text-message="Text_Message"
                    
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
        name: "Media",
        code: "media",
        icon: "M 20.705 9.334 L 20.095 9.334 L 20.095 10.687 L 20.705 10.687 Z M 19.789 9.334 L 19.18 9.334 L 19.18 10.687 L 19.789 10.687 Z M 18.874 9.334 L 18.264 9.334 L 18.264 10.687 L 18.874 10.687 Z M 21.316 9.334 L 21.316 14.742 C 21.316 15.116 21.042 15.418 20.705 15.418 L 17.045 15.418 C 16.707 15.418 16.434 15.116 16.434 14.742 L 16.434 10.687 L 18.264 8.658 L 20.705 8.658 C 21.042 8.658 21.316 8.96 21.316 9.334 Z M 23.898 6.383 L 23.898 16.383 C 23.898 17.493 23.008 18.383 21.898 18.383 L 5.898 18.383 C 4.798 18.383 3.898 17.493 3.898 16.383 L 3.898 4.383 C 3.898 3.273 4.798 2.383 5.898 2.383 L 11.898 2.383 L 13.898 4.383 L 21.898 4.383 C 23.008 4.383 23.898 5.283 23.898 6.383 Z M 22.077 6.102 L 6.077 6.102 L 6.077 16.102 L 22.077 16.102 Z M 2.077 6.102 L 2.077 20.102 L 20.077 20.102 L 20.077 22.102 L 2.077 22.102 C 0.972 22.102 0.077 21.212 0.077 20.102 L 0.077 6.102 Z"
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
    //download: { url: "/localstorage/{storage}/download?path={path}", method: "get" },
    download:{ url: "/localstorage/{storage}{path}", method: "get" },
    upload: { url: "/localstorage/{storage}/upload?path={path}", method: "post" },
    mkdir: { url: "/localstorage/{storage}/mkdir?path={path}", method: "post" },
    delete: { url: "/localstorage/{storage}/delete?path={path}", method: "post" }
};
const IconsSVG={
     ArrowDownLeftBold:"M13.5 21H6V17H13.5C15.43 17 17 15.43 17 13.5S15.43 10 13.5 10H11V14L4 8L11 2V6H13.5C17.64 6 21 9.36 21 13.5S17.64 21 13.5 21Z",
     PlusCircle:"M17,13H13V17H11V13H7V11H11V7H13V11H17M12,2A10,10 0 0,0 2,12A10,10 0 0,0 12,22A10,10 0 0,0 22,12A10,10 0 0,0 12,2Z",
     FolderNew:"M12 12H14V10H16V12H18V14H16V16H14V14H12V12M22 8V18C22 19.11 21.11 20 20 20H4C2.89 20 2 19.11 2 18V6C2 4.89 2.89 4 4 4H10L12 6H20C21.11 6 22 6.89 22 8M20 8H4V18H20V8Z"
};
const fileIcons = {
    sh:"M 12.087 18.491 C 11.578 18.812 10.869 18.973 9.961 18.973 C 9.013 18.973 8.336 18.808 7.928 18.48 C 7.522 18.149 7.317 17.678 7.317 17.062 L 7.317 16.791 L 8.994 16.791 L 8.994 16.986 C 8.994 17.141 9.013 17.283 9.044 17.408 C 9.075 17.535 9.135 17.643 9.219 17.73 C 9.305 17.819 9.42 17.887 9.562 17.936 C 9.707 17.985 9.887 18.011 10.105 18.011 C 10.361 18.011 10.592 17.944 10.795 17.81 C 11.001 17.678 11.104 17.469 11.104 17.191 C 11.104 17.042 11.078 16.913 11.028 16.804 C 10.98 16.695 10.896 16.597 10.779 16.511 C 10.661999999999999 16.424 10.509 16.346 10.318 16.273 C 10.128 16.202 9.893 16.129 9.615 16.054 C 9.242 15.955 8.92 15.845 8.65 15.727 C 8.375 15.609 8.15 15.471 7.966 15.312 C 7.784 15.153 7.649 14.97 7.564 14.762 C 7.479 14.552 7.438 14.312 7.438 14.037 C 7.438 13.379 7.667 12.889 8.124 12.566 C 8.582 12.242 9.213 12.081 10.014 12.081 C 10.385 12.081 10.729 12.113 11.042 12.179 C 11.357 12.244 11.629 12.348 11.858 12.496 C 12.087 12.642 12.267 12.828 12.393 13.053 C 12.52 13.281 12.587 13.552 12.587 13.869 L 12.587 14.056 L 10.98 14.056 C 10.98 13.739 10.909 13.495 10.77 13.322 C 10.631 13.153 10.398 13.067 10.071 13.067 C 9.885 13.067 9.729 13.09 9.608 13.131 C 9.484 13.175 9.384 13.232 9.311 13.304 C 9.238 13.376 9.182 13.461 9.156 13.554 C 9.128 13.647 9.115 13.745 9.115 13.843 C 9.115 14.051 9.167 14.221 9.278 14.361 C 9.387 14.502 9.62 14.63 9.974 14.747 L 11.266 15.195 C 11.585 15.307 11.845 15.424 12.047 15.547 C 12.248 15.668 12.41 15.798 12.53 15.939 C 12.65 16.078 12.732 16.23 12.78 16.399 C 12.827 16.565 12.85 16.755 12.85 16.957 C 12.85 17.66 12.596 18.172 12.087 18.491 Z M 17.477 18.772 L 16.056 18.772 L 16.056 15.81 L 14.398 15.81 L 14.398 18.772 L 12.975999999999999 18.772 L 12.975999999999999 12.097 L 14.398 12.097 L 14.398 14.649000000000001 L 16.056 14.649000000000001 L 16.056 12.097 L 17.477 12.097 Z M 14.434 2.051 L 6.434 2.051 C 5.324 2.051 4.434 2.951 4.434 4.051 L 4.434 20.051 C 4.434 21.161 5.324 22.051 6.434 22.051 L 18.434 22.051 C 19.544 22.051 20.434 21.161 20.434 20.051 L 20.434 8.051 L 14.434 2.051 M 18.384 20.051 L 6.384 20.051 L 6.384 4.051 L 13.384 4.051 L 13.384 9.051 L 18.384 9.051 L 18.384 20.051",
    m3u8:"M 15.239 15.693 L 7.124 15.693 L 7.124 14.8 L 15.239 14.8 Z M 15.239 13.906 L 7.124 13.906 L 7.124 13.013 L 15.239 13.013 Z M 7.124 16.587 L 13.33 16.587 L 13.33 17.48 L 7.124 17.48 Z M 16.67 17.927 L 14.284 19.268 L 14.284 16.587 Z M 20 8 L 20 20 C 20 21.105 19.105 22 18 22 L 6 22 C 4.895 22 4 21.105 4 20 L 4 4 C 4 2.895 4.895 2 6 2 L 14 2 Z M 18 9 L 13 9 L 13 4 L 6 4 L 6 20 L 18 20 Z",
    m3u:"M 15.239 15.693 L 7.124 15.693 L 7.124 14.8 L 15.239 14.8 Z M 15.239 13.906 L 7.124 13.906 L 7.124 13.013 L 15.239 13.013 Z M 7.124 16.587 L 13.33 16.587 L 13.33 17.48 L 7.124 17.48 Z M 16.67 17.927 L 14.284 19.268 L 14.284 16.587 Z M 20 8 L 20 20 C 20 21.105 19.105 22 18 22 L 6 22 C 4.895 22 4 21.105 4 20 L 4 4 C 4 2.895 4.895 2 6 2 L 14 2 Z M 18 9 L 13 9 L 13 4 L 6 4 L 6 20 L 18 20 Z",
    conf:"M6 2C4.89 2 4 2.9 4 4V20C4 21.11 4.89 22 6 22H12V20H6V4H13V9H18V12H20V8L14 2M18 14C17.87 14 17.76 14.09 17.74 14.21L17.55 15.53C17.25 15.66 16.96 15.82 16.7 16L15.46 15.5C15.35 15.5 15.22 15.5 15.15 15.63L14.15 17.36C14.09 17.47 14.11 17.6 14.21 17.68L15.27 18.5C15.25 18.67 15.24 18.83 15.24 19C15.24 19.17 15.25 19.33 15.27 19.5L14.21 20.32C14.12 20.4 14.09 20.53 14.15 20.64L15.15 22.37C15.21 22.5 15.34 22.5 15.46 22.5L16.7 22C16.96 22.18 17.24 22.35 17.55 22.47L17.74 23.79C17.76 23.91 17.86 24 18 24H20C20.11 24 20.22 23.91 20.24 23.79L20.43 22.47C20.73 22.34 21 22.18 21.27 22L22.5 22.5C22.63 22.5 22.76 22.5 22.83 22.37L23.83 20.64C23.89 20.53 23.86 20.4 23.77 20.32L22.7 19.5C22.72 19.33 22.74 19.17 22.74 19C22.74 18.83 22.73 18.67 22.7 18.5L23.76 17.68C23.85 17.6 23.88 17.47 23.82 17.36L22.82 15.63C22.76 15.5 22.63 15.5 22.5 15.5L21.27 16C21 15.82 20.73 15.65 20.42 15.53L20.23 14.21C20.22 14.09 20.11 14 20 14M19 17.5C19.83 17.5 20.5 18.17 20.5 19C20.5 19.83 19.83 20.5 19 20.5C18.16 20.5 17.5 19.83 17.5 19C17.5 18.17 18.17 17.5 19 17.5Z",
    log:"M 17.259 11.102 L 17.259 12.606 L 15.155 12.606 L 15.155 17.118 L 16.208 17.118 L 16.208 14.11 L 17.259 14.11 L 17.259 17.118 C 17.259 17.946 16.786 18.622 16.208 18.622 L 15.155 18.622 C 14.577 18.622 14.103 17.946 14.103 17.118 L 14.103 12.606 C 14.103 11.779 14.577 11.102 15.155 11.102 Z M 7.793 11.102 L 7.793 17.118 L 9.897 17.118 L 9.897 18.622 L 6.741 18.622 L 6.741 11.102 Z M 12.526 11.102 C 13.105 11.102 13.578 11.779 13.578 12.606 L 13.578 17.118 C 13.578 17.946 13.105 18.622 12.526 18.622 L 11.473 18.622 C 10.895 18.622 10.422 17.946 10.422 17.118 L 10.422 12.606 C 10.422 11.779 10.895 11.102 11.473 11.102 Z M 11.473 17.118 L 12.526 17.118 L 12.526 12.606 L 11.473 12.606 Z M 20 8 L 20 20 C 20 21.105 19.105 22 18 22 L 6 22 C 4.895 22 4 21.105 4 20 L 4 4 C 4 2.895 4.895 2 6 2 L 14 2 Z M 18 9 L 13 9 L 13 4 L 6 4 L 6 20 L 18 20 Z",
    zip: "mdi-folder-zip-outline",
    rar: "mdi-folder-zip-outline",
    gz: "mdi-folder-zip-outline",
    htm: "mdi-language-html5",
    html: "mdi-language-html5",
    js: "mdi-nodejs",
    json: "mdi-json",
    py:"mdi-language-python",
    pyc:"M 21.815 7.286 L 21.815 21.387 C 21.815 22.685 20.703 23.737 19.329 23.737 L 4.415 23.737 C 3.042 23.737 1.93 22.685 1.93 21.387 L 1.93 2.586 C 1.93 1.288 3.042 0.236 4.415 0.236 L 14.357 0.236 Z M 19.329 8.461 L 13.116 8.461 L 13.116 2.586 L 4.415 2.586 L 4.415 21.387 L 19.329 21.387 Z M 11.015 11.176 C 11.299 11.176 11.526 10.959 11.526 10.529 C 11.526 10.099 11.299 10.012 11.015 10.012 C 10.74 10.012 10.51 10.099 10.51 10.529 C 10.51 10.959 10.74 11.176 11.015 11.176 Z M 7.973 19.754 C 6.85 19.754 5.94 18.823 5.94 17.674 L 5.94 14.923 C 5.94 13.772 6.85 12.841 7.973 12.841 L 13.047 12.841 C 13.047 12.557 12.819 12.144 12.544 12.144 L 9.496 12.144 L 9.496 10.921 C 9.496 9.773 10.403 8.841 11.526 8.841 L 14.568 8.841 C 15.694 8.841 16.601 9.773 16.601 10.921 L 16.601 13.649 C 16.601 14.8 15.694 15.723 14.568 15.723 L 10.838 15.723 C 9.715 15.723 8.813 16.654 8.813 17.804 L 8.813 19.754 Z M 10.796 17.009 C 10.419 17.009 10.113 17.716 10.113 18.579 C 10.113 19.443 10.419 20.151 10.796 20.151 C 11.169 20.151 11.478 19.443 11.478 18.579 C 11.478 17.716 11.169 17.009 10.796 17.009 Z M 10.796 16.223 C 11.546 16.223 12.162 17.276 12.162 18.579 C 12.162 19.883 11.546 20.936 10.796 20.936 C 10.044 20.936 9.43 19.883 9.43 18.579 C 9.43 17.276 10.044 16.223 10.796 16.223 Z M 14.894 16.223 L 15.577 16.223 L 15.577 20.151 L 16.259 20.151 L 16.259 20.936 L 14.211 20.936 L 14.211 20.151 L 14.894 20.151 L 14.894 17.009 L 14.211 17.4 L 14.211 16.616 Z",
    hex:"mdi-file-code-outline",
    go:"mdi-language-go",
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
    other: "mdi-file-outline",
    
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
        IconsToolbar: { type: Object, default: () => IconsSVG },
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