<template>
    <div id="graph" ref="div_ref">
        <v-network-graph ref="graph_ref" v-model:selected-nodes="graphData.selectedNodes"
            v-model:selected-edges="graphData.selectedEdges" :nodes="graphData.nodes"
            :edges="graphData.edges" :layouts="graphData.layouts"
            :configs="graphData.configs" :event-handlers="eventHandlers" />
    </div>
    <DataPanelVue @pauseKeyDown="pauseKeyDown" @resumeKeyDown="resumeKeyDown"
        :data_panel="dataPanel" @updatedDataPanel="updatedDataPanel"></DataPanelVue>
    <ControlPanelVue @home="home" @save-locations="saveLocations" @customkeydown="keydown" ref="control_panel_ref">
    </ControlPanelVue>
</template>
<script lang="ts">
import axios from "axios";
import DataPanelVue from "../components/DataPanel.vue";
import ControlPanelVue from "../components/ControlPanel.vue";
import { defineComponent, reactive, ref } from "vue";
import graphData from "../graphData"
var dataPanel: any = ref({});

const project_id: any = ref("");
const graph_ref: any = ref();
const control_panel_ref: any = ref();
var pause_key_down: boolean = false;

function delete_node(node: any) {
    const request_url = `${import.meta.env.VITE_API_URL}` +
        `/project/${project_id.value}` +
        `/resource/${node.substring(4)}` +
        `/delete`
    axios.post(request_url)
        .then(response => {
            delete graphData.nodes.value[node]
            delete graphData.layouts.value.nodes[node]
        })
}
function delete_edge(edge: any) {
    const request_url = `${import.meta.env.VITE_API_URL}` +
        `/project/${project_id.value}` +
        `/relationship/${edge}` +
        `/delete`
    axios.post(request_url)
        .then(response => {
            delete graphData.edges.value[edge]
        })
}
function add_node(raw: any) {
    graphData.nodes.value[`node${raw.uid}`] = raw
    if('pos_x' in raw && 'pos_y' in raw){
        graphData.layouts.value.nodes[`node${raw.uid}`] = {
            'x': raw['pos_x'],
            'y': raw['pos_y']
        }
    }
}
function add_edge(s: string, t: string, uid: any) {
    graphData.edges.value[uid] =
    {
        source: s,
        target: t,
        uid: uid
    }
}
var selected_nodes = ref([]);
var selected_edges = ref([]);


function add_edge_raw(raw: any) {
    add_edge(`node${raw[0]}`, `node${raw[2]}`, raw[1].uid)
}

function get_items() {
    const request_url = import.meta.env.VITE_API_URL + `/project/${project_id.value}`
    axios
        .get(request_url)
        .then(response => {
            graphData.nodes.value = {}
            graphData.edges.value = {}
            console.log(response.data)
            var edge_queue: Array<Object> = []
            for (const item of response.data.items) {
                if (item instanceof Array) {
                    // only propagate edge info afer all node info has been done
                    // this is because edge info depends on node_uid_name_mapping 
                    edge_queue.push(item)
                } else {
                    add_node(item)
                }
            }
            for (const item of edge_queue) {
                add_edge_raw(item)
            }
        })
}
export default defineComponent({
    name: "Graph",
    components: {
        DataPanelVue,
        ControlPanelVue
    },
    data() {
        return {
            graph_ref,
            project_id,
            graphData,
            dataPanel,
            eventHandlers: {
                // wildcard: capture all events
                "*": (type: string, event: any) => {
                    console.log(type, event)
                    if (type == 'node:pointerover') {
                        dataPanel.value = graphData.nodes.value[event.node]
                    } else if (type == 'view:click') {
                        this.handle_view_click(event)
                    } else if (type == 'node:select') {
                        this.handle_node_select(event)
                    } else if (type == 'node:click') {
                        this.handle_node_click(event)
                    } else if (type == 'edge:select') {
                        this.handle_edge_select(event)
                    }
                },
            }
        }
    },
    methods: {
        updatedDataPanel: function(){
            dataPanel.value = graphData.nodes.value[`node${dataPanel.value.uid}`] 
        },
        handle_view_click: function (event: any) {
            const selected_mode = (this.$store.state as any).selected_mode
            if (selected_mode == 'add-node') {
                const request_url = `${import.meta.env.VITE_API_URL}` +
                    `/project/${project_id.value}` +
                    `/new?item=node`
                axios
                    .post(request_url)
                    .then(response => {
                        this.add_node_with_mouse(response.data, event.event);
                    });
                (this.$store.state as any).selected_mode = 'move'
            } else if (selected_mode == 'add-edge') {
                (this.$store.state as any).edge_source_node = null;
            }
        },
        handle_node_click: function (event: any) {
            const selected_mode = (this.$store.state as any).selected_mode
            if (selected_mode == 'add-edge') {
                if ((this.$store.state as any).edge_source_node) {
                    this.create_edge((this.$store.state as any).edge_source_node, event.node);
                    (this.$store.state as any).selected_mode = 'move'
                }
            } else {
                (this.$store.state as any).edge_source_node = event.node
            }
        },
        create_edge: function (s: any, t: any) {
            const request_url = `${import.meta.env.VITE_API_URL}` +
                `/project/${project_id.value}` +
                `/new?item=relationship` +
                `&a_id=${s.substring(4)}&b_id=${t.substring(4)}`
            axios.post(request_url)
                .then(response => {
                    graphData.edges.value[response.data.uid] = {
                        source: s,
                        target: t,
                        uid: response.data.uid
                    }
                });
            (this.$store.state as any).edge_source_node = null
        },
        handle_node_select: function (event: any) {
            const selected_mode = (this.$store.state as any).selected_mode
            if (selected_mode == 'add-edge') {
                if (event.length == 1) {
                    (this.$store.state as any).edge_source_node = event[0]
                }else {
                    (this.$store.state as any).edge_source_node = null
                }
            }
            selected_nodes.value = event
        },
        handle_edge_select: function (event: any) {
            selected_edges.value = event
        },
        home: function () {
            const inf = 10000000000000000;
            var minX = inf, minY = inf, maxX = -inf, maxY = -inf;
            for (const i in graphData.layouts.value.nodes) {
                const node = graphData.layouts.value.nodes[i]
                minX = Math.min(minX, node.x)
                maxX = Math.max(maxX, node.x)
                minY = Math.min(minY, node.y)
                maxY = Math.max(maxY, node.y)
            }
            const padding = 100;
            (this.$refs.graph_ref as any).setViewBox({
                left: minX - padding,
                top: minY - padding,
                right: maxX + padding * 5,
                bottom: maxY + padding,
            });
        },
        saveLocations: function(){
            const node_pos_raw = (this.$refs.graph_ref as any).layouts.nodes
            const node_pos : any = {}
            for(var n in node_pos_raw){
                node_pos[n.substring(4)] = {
                    x: node_pos_raw[n].x,
                    y: node_pos_raw[n].y,
                }
            }
            console.log(node_pos)
            const request_url = `${import.meta.env.VITE_API_URL}` +
                `/project/${project_id.value}` +
                `/positions/update`
            axios.post(request_url, node_pos)
        },
        get_items,
        add_node_with_mouse: function (raw: any, e: any) {
            const point = { x: e.offsetX, y: e.offsetY }
            const svgPoint = (this.$refs.graph_ref as any).translateFromDomToSvgCoordinates(point);
            graphData.layouts.value.nodes[`node${raw.uid}`] = svgPoint
            add_node(raw)
        },
        keydown: function (e: any) {
            if (pause_key_down) return;
            if (e.key == 'e') {
                this.$store.commit('update_selected_mode', 'add-edge')
            } else if (e.key == 'v') {
                this.$store.commit('update_selected_mode', 'add-node')
            } else if (e.key == 'a') {
                this.$store.commit('update_selected_mode', 'move')
            } else if (e.key == 'Backspace' || e.key == 'Delete') {
                if (selected_nodes.value.length > 0 && confirm(`Delete ${selected_nodes.value.length} nodes?`)) {
                    for (const node of selected_nodes.value) {
                        delete_node(node)
                    }
                }
                if (selected_edges.value.length > 0 && confirm(`Delete ${selected_edges.value.length} edges?`)) {
                    for (const edge of selected_edges.value) {
                        delete_edge(edge)
                    }
                }
            }
        },
        pauseKeyDown: function () { pause_key_down = true; },
        resumeKeyDown: function () { pause_key_down = false; }
    },
    mounted() {
        axios.defaults.headers.common['X-User'] = this.$store.state.kratos_user_id
        project_id.value = this.$route.params.id
        get_items()
    },
    beforeUnmount() {
        // so that the next time we open the graph,
        // there won't be a flicker from the old graph data
        graphData.nodes.value = {}
        graphData.edges.value = {}
    }
})
</script>