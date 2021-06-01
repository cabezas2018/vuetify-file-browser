<template>
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
</template>
<script>
export default {
    data: () => ({
        snackbar: false,
        resolve: null,
        reject: null,
        messageText: null,
       
        options: {
            color: "green",
            width: 300,
            position:"right",
            messageText:""
        }
    }),
    methods: {
        open(title, message, options) {
            this.snackbar = true;
            this.options = Object.assign(this.options, options);
            return new Promise((resolve, reject) => {
                this.resolve = resolve;
                this.reject = reject;
            });
        },
        agree() {
            this.resolve(true);
            this.snackbar = false;
        },
        cancel() {
            this.resolve(false);
            this.snackbar = false;
        },
        toast(message = "", type = "success") {
            //console.log("toast", message, "type:"+type,"position:"+position);
            this.snackbar = true;
            this.options.color =
            type == "error"
                ? "red"
                : type == "success"
                ? "green"
                : type == "warning"
                ? "orange"
                : "cyan";
            // this.options.position.right   =   position == "right" ? true:false 
            // this.options.position.left =  position == "left" ? true:false
            // this.options.position.centered = position == "centered" ? true:false
            //console.log(this.options.position)
            this.options.messageText = message;
            return new Promise((resolve, reject) => {
                this.resolve = resolve;
                this.reject = reject;
            });
            
        },
    }
    
}
</script>