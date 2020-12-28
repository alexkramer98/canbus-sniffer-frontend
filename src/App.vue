<template>
    <div>
        <div class="d-flex justify-content-between p-2 bg-light">
            <p class="mt-3">Connected to Arduino: <b-badge :variant="isConnected ? 'success' : 'danger'">{{isConnected ? 'Yes': 'No'}}</b-badge></p>
            <p class="mt-3">CAN initialized: <b-badge :variant="isInitialized ? 'success' : 'danger'">{{isInitialized ? 'Yes': 'No'}}</b-badge></p>
            <p class="mt-3">Seen data: <b-badge :variant="hadData ? 'success' : 'danger'">{{hadData ? 'Yes': 'No'}}</b-badge></p>
        </div>
        <div class="d-flex justify-content-between">
            <b-input-group prepend="Filter (id):">
                <b-form-input class="rounded-0" placeholder="Optional" v-model="filterText"></b-form-input>
            </b-input-group>
            <b-input-group prepend="Port:">
                <b-select :options="ports" v-model="selectedPort"/>
            </b-input-group>
            <b-input-group prepend="CAN baudrate:">
                <b-select v-model="selectedBaudrate">
                    <b-select-option v-for="baudrate of canBaudrates" :value="baudrate" :key="baudrate">{{baudrate}}Kbps</b-select-option>
                </b-select>
            </b-input-group>
            <b-btn variant="success" class="rounded-0" @click="onConnect" :disabled="!(selectedPort && selectedBaudrate)">Connect</b-btn>
        </div>
        <table class="table table-light text-center small">
            <thead class="thead-dark">
            <tr>
                <th>ID</th>
                <th v-for="i in 8" :key="i">{{i}}</th>
            </tr>
            </thead>
            <tbody>
            <tr v-for="message of messages" :key="message.id">
                <td>0x{{ message.id }}</td>
                <td v-for="(byte, index) in message.msg" :key="message.id + '-' + index">0x{{ byte }}</td>
            </tr>
            </tbody>
        </table>
    </div>
</template>

<script>
const SerialPort = require('serialport');
const ReadLine = require('@serialport/parser-readline');

export default {
    name: 'App',
    data() {
        return {
            ports: [],
            isConnected: false,
            isInitialized: false,
            hadData: false,
            selectedPort: null,
            selectedBaudrate: null,
            filterText: null,
            canBaudrates: [5, 10, 20, 25, 33, 50, 80, 95, 100, 125, 200, 250, 500, 666, 1000],
            arduino: null,
            log: [],
            messages: [],
        }
    },
    created() {
        SerialPort.list().then(data => {
            data.forEach(item => {
               if (item.manufacturer && item.manufacturer.includes('arduino')) {
                   this.ports.push(item.path);
               }
            });
            if (!this.ports.length) {
                this.$bvToast.toast('Could not find any Arduino connected to this computer. Please check your connection and restart the program.', {
                    title: 'Error',
                    variant: 'danger',
                })
            } else {
                this.selectedPort = this.ports[0];
            }
        }).catch(error => {
            this.$bvToast.toast('Unable to fetch available serial ports. Error: ' + error, {
                title: 'Error',
                variant: 'danger',
            });
        });
    },
    // watch: {
    //     messages() {
    //         console.log(this.messages);
    //     }
    // },
    methods: {
        onConnect() {
            this.isConnected = false;
            this.isInitialized = false;
            this.hadData = false;
            if (this.arduino) {
                this.arduino.close();
            }
            this.$bvToast.toast('Attempting to connect to Arduino.', {
                title: 'Connecting...',
                variant: 'primary',
            });
            this.arduino = new SerialPort(this.selectedPort, {
                baudRate: 115200
            });
            this.arduino.on('close', () => {
                if (this.isConnected) {
                    this.$bvToast.toast('Connection was lost!', {
                        title: 'Disconnected',
                        variant: 'warning',
                    });
                }
                this.isConnected = false;
            });
            setTimeout(() => {
                if (!this.isConnected) {
                    this.$bvToast.toast('Could not talk to the Arduino. Is the device in use by another process?', {
                        title: 'Device timeout!',
                        variant: 'danger',
                    });
                }
            }, 5000)
            const parser = this.arduino.pipe(new ReadLine({ delimiter: '\n' }));
            parser.on('data', data => this.handleSerialData(data.trim()));
        },
        handleSerialData(data) {
            if (data === 'Ready.') {
                this.$bvToast.toast('Connected! Now attempting to initialize CAN.', {
                    title: 'Connected',
                    variant: 'success',
                });
                this.isConnected = true;
                this.arduino.write(this.selectedBaudrate.toString());
            } else if (data === 'CAN Initialized! Sniffing now.') {
                this.$bvToast.toast('CAN successfully initiated!. Now sniffing data.', {
                    title: 'Initiated!',
                    variant: 'success',
                });
                this.isInitialized = true;
            } else if (data === 'CAN not initialized. Check settings!') {
                this.$bvToast.toast('CAN could not be initialized. Please check your connections and your CS pin in the Arduino sketch.', {
                    title: 'CAN Error!',
                    variant: 'danger',
                });
            } else if (data.includes(':')) {
                if (!this.isInitialized) {
                    return;
                }
                this.hadData = true;
                const splitMsg = data.split(':');
                const id = splitMsg[0];
                const msg = splitMsg[1].split(',');

                const existingIndex = this.messages.findIndex(item => item.id === id);

                if (existingIndex !== -1) {
                    this.$set(this.messages, existingIndex, {
                        id,
                        msg
                    });
                } else {
                    this.messages.push({
                        id,
                        msg
                    })
                }
            }
        }
    }
}
</script>

<style>

</style>
