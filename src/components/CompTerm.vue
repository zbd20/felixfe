<template>
    <el-dialog :visible.sync="v"
               :title="obj.user + '@' + obj.host"
               @opened="doOpened"
               @open="doOpen"
               @close="doClose"
               center
               fullscreen
               custom-class="felix-dialog"
    >

        <div ref="terminal" class="felix-xterm"></div>

    </el-dialog>
</template>

<script>
    import {Terminal} from "xterm";
    import * as fit from "xterm/lib/addons/fit/fit";
    import {Base64} from "js-base64";
    import * as webLinks from "xterm/lib/addons/webLinks/webLinks";
    import * as search from "xterm/lib/addons/search/search";
    import "xterm/dist/xterm.css"
    import config from "@/config/config"

    let defaultTheme = {
        foreground: "#ffffff",
        background: "#1b212f",
        cursor: "#ffffff",
        selection: "rgba(255, 255, 255, 0.3)",
        black: "#000000",
        brightBlack: "#808080",
        red: "#ce2f2b",
        brightRed: "#f44a47",
        green: "#00b976",
        brightGreen: "#05d289",
        yellow: "#e0d500",
        brightYellow: "#f4f628",
        magenta: "#bd37bc",
        brightMagenta: "#d86cd8",
        blue: "#1d6fca",
        brightBlue: "#358bed",
        cyan: "#00a8cf",
        brightCyan: "#19b8dd",
        white: "#e5e5e5",
        brightWhite: "#ffffff"
    };
    let bindTerminalResize = (term, websocket) => {
        let onTermResize = size => {
            websocket.send(
                JSON.stringify({
                    type: "resize",
                    rows: size.rows,
                    cols: size.cols
                })
            );
        };
        // register resize event.
        term.on("resize", onTermResize);
        // unregister resize event when WebSocket closed.
        websocket.addEventListener("close", function () {
            term.off("resize", onTermResize);
        });
    };
    let bindTerminal = (term, websocket, bidirectional, bufferedTime) => {
        term.write('\r\nwelcome to https://github.com/dejavuzhou/felix\n\r\n');
        term.socket = websocket;
        let messageBuffer = null;
        let handleWebSocketMessage = function (ev) {
            if (bufferedTime && bufferedTime > 0) {
                if (messageBuffer) {
                    messageBuffer += ev.data;
                } else {
                    messageBuffer = ev.data;
                    setTimeout(function () {
                        term.write(messageBuffer);
                    }, bufferedTime);
                }
            } else {
                term.write(ev.data);
            }
        };

        let handleTerminalData = function (data) {
            websocket.send(
                JSON.stringify({
                    type: "cmd",
                    cmd: Base64.encode(data) // encode data as base64 format
                })
            );
        };

        websocket.onmessage = handleWebSocketMessage;
        if (bidirectional) {
            term.on("data", handleTerminalData);
        }

        // send heartbeat package to avoid closing webSocket connection in some proxy environmental such as nginx.
        let heartBeatTimer = setInterval(function () {
            websocket.send(JSON.stringify({type: "heartbeat", data: ""}));
        }, 20 * 1000);

        websocket.addEventListener("close", function () {
            websocket.removeEventListener("message", handleWebSocketMessage);
            term.off("data", handleTerminalData);
            delete term.socket;
            clearInterval(heartBeatTimer);
        });
    };
    export default {
        props: {obj: {type: Object, require: true}, visible: Boolean},
        name: "CompTerm",
        data() {
            return {
                isFullScreen: false,
                searchKey: "",
                v: this.visible,
                ws: null,
                term: null,
                thisV: this.visible
            };
        },
        watch: {
            visible(val) {
                this.v = val;//新增result的watch，监听变更并同步到myResult上
            }
        },
        computed: {
            wsUrl() {
                let token = localStorage.getItem('token');
                return `${config.wsBase}/api/ws/${this.obj.ID || 0}?cols=${this.term.cols}&rows=${this.term.rows}&_t=${token}`
            }
        },

        methods: {

            onWindowResize() {
                //console.log("resize")
                this.term.fit(); // it will make terminal resized.
            },
            doLink(ev, url) {
                if (ev.type === 'click') {
                    window.open(url)
                }
            },
            doClose() {
                window.removeEventListener("resize", this.onWindowResize);
                // term.off("resize", this.onTerminalResize);
                if (this.ws) {
                    this.ws.close()
                }
                if (this.term) {
                    this.term.dispose()
                }
                this.$emit('pclose', false)//子组件对openStatus修改后向父组件发送事件通知
            },
            doOpen() {

            },
            doOpened() {
                Terminal.applyAddon(fit);
                Terminal.applyAddon(webLinks);
                Terminal.applyAddon(search);
                this.term = new Terminal({
                    //rows: 20,
                    fontSize: 18,
                    cursorBlink: true,
                    cursorStyle: 'bar',
                    bellStyle: "sound",
                    theme: defaultTheme
                });
                this.term.open(this.$refs.terminal);
                this.term.fit(); // first resizing
                this.term.webLinksInit(this.doLink);
                // term.on("resize", this.onTerminalResize);
                window.addEventListener("resize", this.onWindowResize);
                this.ws = new WebSocket(this.wsUrl);
                // this.ws.onerror = event => {
                //     console.log(event)
                //     this.$message.error('ssh ws failed please retry or check your auth config');
                // };
                this.ws.onclose = (ce) => {
                    //console.log(ce)
                    this.term.setOption("cursorBlink", false);
                    if (ce.code !== 1005) {
                        this.$notify.error({
                            title: `code ${ce.code}`,
                            message: ce.reason,
                        });
                        this.doClose();
                    }
                };

                bindTerminal(this.term, this.ws, true, -1);
                bindTerminalResize(this.term, this.ws);
            },

        },


    }
</script>

<style>
    .felix-xterm {
        height: calc(100vh - 60px) !important;
    }

    .felix-dialog {
        background-color: #1b212f !important;
        margin: 0 !important;

    }

    .felix-dialog .el-dialog__body {
        /*border: white solid 1px!important;*/
        padding: 0 !important;

    }

    .felix-dialog .el-dialog__title {
        color: white;
        font-weight: bold;
    }

    .felix-dialog .el-dialog__headerbtn {
        font-size: 2rem !important;
        color: white;
        font-weight: bold !important;
    }
</style>