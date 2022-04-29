<template>
    <div class="card">
        <div class="card-header text-center">WELCOME TO LEDIS</div>
        <div class="card-body">
            <div :key="element.id" v-for="element in conversation">
                <div>> {{ element.req }}</div>
                <div :class="element.res.substring(0, 5).toLowerCase()">{{ element.res }}</div>
            </div>
        </div>
        <div class="card-footer text-muted containter-fluid" style="background-color: black">
            <div class="row">
                <form @submit="onSubmit">
                    <div class="input-group">
                        <div class="input-group-prepend">
                            <div class="input-group-text input-command">
                                >Ledis:</div>
                        </div>
                        <input type="text" class="form-control input-command" id="inlineFormInputGroupUsername2"
                            name="input" v-model="input" />
                    </div>
                </form>

            </div>
        </div>
        <div id="output" style="display:none"></div>
    </div>
</template>
<style scoped>
.card {
    height: 100vh
}

.error {
    background-color: red;
    color: white
}

.card-header {
    background-color: palevioletred;
    border: 1px solid black;
    color: whitesmoke;
}

.input-command {
    color: white;
    background-color: black;
    border: 0
}
</style>
<script>
import CryptoJS from 'crypto-js';
export default {
    name: 'CommandBox',
    data() {
        return {
            input: '',
            conversation: [],
            stored_data: {},
            expiration: {},
            secret: "abc1234!",
            interval_id: {}
        }
    },
    mounted() {
        var get_data = localStorage.getItem("stored_data")
        console.log(get_data)
        if (get_data) {
            this.stored_data = JSON.parse(get_data)
            localStorage.removeItem("stored_data")
        }
        window.addEventListener('beforeunload', this.beforWindowClose)
    },
    methods: {
        onSubmit(e) {
            e.preventDefault()
            if (!this.input) {
                alert('Please input command')
                return
            }
            let input_split = this.input.split(" ").filter((item) => {
                return item !== ""
            })
            var req_res = { id: this.conversation.length + 1, req: this.input }

            if (input_split[0].toUpperCase() === "SET") {
                if (this.handleCommandError("SET", input_split)) {
                    this.set(input_split[1], input_split[2])
                    req_res["res"] = "OK"
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }

            }
            else if (input_split[0].toUpperCase() === "GET") {
                if (this.handleCommandError("GET", input_split)) {
                    if (this.validateKey(input_split[1])) {
                        if (!this.handleTypeError(input_split[1])) {
                            req_res["res"] = this.get(input_split[1])
                        } else {
                            req_res["res"] = "null"
                        }
                    } else {
                        req_res["res"] = "null"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }

            }
            else if (input_split[0].toUpperCase() === "SADD") {
                if (this.handleCommandError("SADD", input_split)) {
                    if (this.handleTypeError(input_split[1])) {
                        for (let index = 2; index < input_split.length; index++) {
                            req_res["res"] = "Current length of this set: " + this.sadd(input_split[1], input_split[index])
                        }
                    } else {
                        req_res["res"] = "ERROR: Can't add value to string"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }

            }
            else if (input_split[0].toUpperCase() === "SREM") {
                if (this.handleCommandError("SREM", input_split)) {
                    if (this.validateKey(input_split[1])) {
                        if (this.handleTypeError(input_split[1])) {
                            for (let index = 2; index < input_split.length; index++) {
                                req_res["res"] = "Current length of this set: " + this.srem(input_split[1], input_split[index])
                            }
                        } else {
                            req_res["res"] = "ERROR: Can't remove value from string"
                        }
                    } else {
                        req_res["res"] = "ERROR: Can't find this key"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }
            else if (input_split[0].toUpperCase() === "SMEMBERS") {
                if (this.handleCommandError("SMEMBERS", input_split)) {
                    if (this.validateKey(input_split[1])) {
                        if (this.handleTypeError(input_split[1])) {
                            const result = this.smembers(input_split[1])
                            let res = ''
                            for (let index = 0; index < result.length; index++) {
                                res += result[index] + ' '
                            }
                            req_res["res"] = res
                        } else {
                            req_res["res"] = "ERROR: Can't get members from string"
                        }
                    } else {
                        req_res["res"] = "null"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }

            else if (input_split[0].toUpperCase() === "SINTER") {
                if (this.handleCommandError("SINTER", input_split)) {
                    const res = this.sinter(input_split.splice(1))
                    if (res !== false) {
                        req_res["res"] = this.sinter(input_split.splice(1))
                    } else {
                        req_res["res"] = "ERROR: Can't intersect with string or invalid key"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }

            else if (input_split[0].toUpperCase() === "KEYS") {
                if (this.handleCommandError("KEYS", input_split)) {
                    if (this.keys().length === 0) {
                        req_res["res"] = "None"
                    } else {
                        req_res["res"] = this.keys().toString()
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }

            else if (input_split[0].toUpperCase() === "DEL") {
                if (this.handleCommandError("DEL", input_split)) {
                    this.del(input_split[1])
                    req_res["res"] = "OK"
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }
            else if (input_split[0].toUpperCase() === "EXPIRE") {
                if (this.handleCommandError("EXPIRE", input_split)) {
                    if (this.validateKey(input_split[1])) {
                        if (isNaN(parseInt(input_split[2]))) {
                            req_res["res"] = "ERROR: Invalid number of seconds"
                        } else {
                            this.expire(input_split[1], input_split[2])
                            req_res["res"] = "This key is expired in " + input_split[2] + " seconds"
                        }
                    } else {
                        req_res["res"] = "ERROR: Can't find this key"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }
            else if (input_split[0].toUpperCase() === "TTL") {
                if (this.handleCommandError("TTL", input_split)) {
                    if (this.validateKey(input_split[1])) {
                        req_res["res"] = this.ttl(input_split[1])
                    } else {
                        req_res["res"] = "ERROR: Can't find this key"
                    }
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            }
            else if (input_split[0].toUpperCase() === "SAVE") {
                if (this.handleCommandError("SAVE", input_split)) {
                    this.save()
                    req_res["res"] = "OK"
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }

            }
            else if (input_split[0].toUpperCase() === "RESTORE") {
                if (this.handleCommandError("RESTORE", input_split)) {
                    req_res["res"] = this.restore()
                } else {
                    req_res["res"] = "ERROR: Invalid Command"
                }
            } else {
                req_res["res"] = "ERROR: Invalid Command"
            }
            this.conversation.push(req_res)
            this.input = ''
        },

        beforWindowClose() {
            const string_result = JSON.stringify(JSON.parse(JSON.stringify(this.stored_data)))
            localStorage.setItem("stored_data", string_result)
        },

        set(key, value) {
            this.stored_data[key] = value
        },

        get(key) {
            return this.stored_data[key]
        },

        sadd(key, value) {
            if (!this.stored_data[key]) {
                this.stored_data[key] = []
            }
            this.stored_data[key].push(value)
            return this.stored_data[key].length
        },

        srem(key, value) {
            this.stored_data[key] = this.stored_data[key].filter((element) => element !== value)
            return this.stored_data[key].length
        },

        smembers(key) {
            return this.stored_data[key]
        },
        sinter(key) {
            let result = []
            if (!this.validateKey(key[0])) {
                return false
            }
            if (!this.handleTypeError(key[0])) {
                return false
            }
            const element = this.stored_data[key[0]]
            element.forEach(ele => {
                let count = 0
                for (let index = 1; index < key.length; index++) {
                    const key_value = this.stored_data[key[index]];
                    if (!this.validateKey(key[index])) {
                        return false
                    }
                    if (!this.handleTypeError(key[index])) {
                        return false
                    }
                    if (key_value.includes(ele)) {
                        count += 1
                    }
                }
                if (count == key.length - 1) {
                    result.push(ele)
                }
            });
            return result
        },

        keys() {
            return Object.keys(this.stored_data)
        },

        del(key) {
            delete this.stored_data[key]
        },
        expire: function (key, seconds) {
            var currentTime = Date.now()
            if (this.interval_id[key]) {
                clearInterval(this.interval_id[key])
            }
            let interval = setInterval(() => {
                console.log((seconds - getTimeLeft()) + "s")
                this.interval_id[key] = interval
                this.expiration[key] = (seconds - getTimeLeft()) + "s"
                if (getTimeLeft() >= seconds) {
                    this.del(key)
                    delete this.expiration[key]
                    delete this.interval_id[key]
                    clearInterval(interval)
                }
            }, 1000);

            function getTimeLeft() {
                return Math.ceil((Date.now() - currentTime) / 1000);
            }
        },
        ttl(key) {
            if (!this.expiration[key]) {
                return "null"
            }
            return this.expiration[key]
        },
        save() {
            const encrypted_data = CryptoJS.AES.encrypt(JSON.stringify(this.stored_data), this.secret).toString();
            localStorage.setItem("snapshot", encrypted_data)
        },
        restore() {
            const enc_data = localStorage.getItem("snapshot")
            try {
                this.stored_data = JSON.parse(CryptoJS.AES.decrypt(enc_data, this.secret).toString(CryptoJS.enc.Utf8));
                return "SUCCESSFULLY RESTORED"
            } catch (error) {
                return "ERROR: Your snapshot database is changed"
            }
        },
        validateKey(key) {
            if (!this.stored_data[key]) {
                return false
            }
            return true
        },
        handleTypeError(key) {
            if (typeof this.stored_data[key] === 'string') {
                return false
            }
            return true
        },
        handleCommandError(command, input) {
            if ((command === "SET" || command === "EXPIRE") && input.length != 3) {
                return false
            }
            if ((command === "GET" || command === "SMEMBERS" || command === "DEL" || command === "TTL") && input.length != 2) {
                return false
            }
            if ((command === "SADD" || command === "SREM") && input.length < 3) {
                return false
            }
            if (command === "SINTER" && input.length < 2) {
                return false
            }
            if ((command === "SAVE" || command === "RESTORE" || command === "KEYS") && input.length > 1) {
                return false
            }
            return true
        }
    },
}
</script>