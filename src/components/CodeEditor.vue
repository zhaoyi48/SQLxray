<template>
    <el-row>
        <el-col :span="12">
            <codemirror v-model="code" placeholder="Code goes here..." :style="{ height: '90vh' }" :autofocus="true"
                :indent-with-tab="true" :tab-size="4" :extensions="extensions" @ready="handleReady"
                @change="xray($event)" />
        </el-col>
        <el-col :span="12">
            <SqlFlow />
        </el-col>
    </el-row>
</template>

<script setup>
import { ref, shallowRef, provide } from 'vue'
import { Codemirror } from 'vue-codemirror'
import { sql } from '@codemirror/lang-sql'
import { oneDark } from '@codemirror/theme-one-dark'
import SqlFlow from './SqlFlow.vue'
import { Parser } from 'node-sql-parser'

const activities = ref([]);
provide('map', activities);
const infoTree = ref({});
const yBias = ref(1);

const code = ref(`select * from test;`)
const extensions = [sql(), oneDark]

// Codemirror EditorView instance ref
const view = shallowRef()
const handleReady = (payload) => {
    view.value = payload.view
}

// Status is available at all times via Codemirror EditorView
const getCodemirrorStates = () => {
    const state = view.value.state
    const ranges = state.selection.ranges
    const selected = ranges.reduce((r, range) => r + range.to - range.from, 0)
    const cursor = ranges[0].anchor
    const length = state.doc.length
    const lines = state.doc.lines
    // more state info ...
    // return ...
}

function xray(sqls) {
    this.infoTree = {};
    this.yBias = 1;
    this.activities.length = 0;
    sqls.split(';').forEach(sql => {
        if (sql.trim() != '') {
            try {
                const parser = new Parser();
                const ast = parser.astify(sql);
                console.log('change', ast);

                if (ast.type == 'select') {
                    this.selectAST(ast, null,true)
                }

                if (ast.type == 'insert') {
                    this.insertAST(ast)
                }


            } catch (error) {
                this.activities.length = 0;
                console.error(error);
            }
            console.log('change', this.activities);
        }
    });

}

function selectAST(ast, exLink,linkAnimated) {
    var maxXbias = 0;
    ast.from.forEach(data => {
        const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        if (this.infoTree[`${tablename}`]) {
            if (maxXbias < this.infoTree[`${tablename}`].xBias) {
                maxXbias = this.infoTree[`${tablename}`].xBias;
            }
        }
    });
    let bf='';
    ast.from.forEach(data => {
        if (data.expr == undefined) {
            const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
            bf=tablename;
            if (!this.infoTree[`${tablename}`]) {
                this.infoTree[`${tablename}`] = {
                    xBias: `${++maxXbias}`,
                    yBias: `${this.yBias}`,
                    sourceTable: []
                }
                const newNode = {
                    id: tablename,
                    label: `${tablename}`,
                    position: { x: 0, y: 0 },
                }
                newNode.position = { x: this.infoTree[`${tablename}`].xBias * 100, y: this.infoTree[`${tablename}`].yBias * 100 }
                this.activities.push(newNode);
                this.yBias++;
            }
        } else {
            if (data.expr.ast.type == 'select') {
                if(exLink==null){
                    exLink=bf;
                }
                this.selectAST(data.expr.ast, exLink,linkAnimated);
            }
            // if (ast.type == 'insert') {
            //     insertAST(ast)
            // }

        }

    });

    this.genEdge(ast, true, exLink,linkAnimated);

}

function insertAST(ast) {
    var maxXbias = 0;
    ast.table.forEach(data => {
        const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        if (this.infoTree[`${tablename}`]) {
            if (maxXbias < this.infoTree[`${tablename}`].xBias) {
                maxXbias = this.infoTree[`${tablename}`].xBias;
            }
        }
    });

    ast.table.forEach(data => {
        const baseTablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        const exLink = baseTablename;
        if (!this.infoTree[`${baseTablename}`]) {
            this.infoTree[`${baseTablename}`] = {
                xBias: `${++maxXbias}`,
                yBias: `${this.yBias}`,
                sourceTable: []
            }
            const newNode = {
                id: baseTablename,
                label: `${baseTablename}`,
                position: { x: 0, y: 0 },
            }
            newNode.position = { x: this.infoTree[`${baseTablename}`].xBias * 100, y: this.infoTree[`${baseTablename}`].yBias * 100 }
            this.activities.push(newNode);
            this.yBias++;
        }
        ast.values.from.forEach(data => {
            if (data.expr == undefined) {

                const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
                if (!this.infoTree[`${tablename}`]) {
                    this.infoTree[`${tablename}`] = {
                        xBias: `${++maxXbias}`,
                        yBias: `${this.yBias}`,
                        sourceTable: []
                    }
                    const newNode = {
                        id: tablename,
                        label: `${tablename}`,
                        position: { x: 0, y: 0 },
                    }
                    newNode.position = { x: this.infoTree[`${tablename}`].xBias * 100, y: this.infoTree[`${tablename}`].yBias * 100 }
                    this.activities.push(newNode);
                    this.infoTree[`${baseTablename}`].sourceTable.push(tablename);
                    const linkEdge = {
                        id: `${baseTablename}-${tablename}`,
                        source: `${baseTablename}`,
                        target: `${tablename}`,
                        style: (edge) => {
                            if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                            return { stroke: '#10b981', strokeWidth: 3 }
                        },
                        // animated: true,
                    }
                    this.activities.push(linkEdge);
                    this.yBias++;
                } else {
                    const linkEdge = {
                        id: `${baseTablename}-${tablename}`,
                        source: `${baseTablename}`,
                        target: `${tablename}`,
                        style: (edge) => {
                            if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                            return { stroke: '#10b981', strokeWidth: 3 }
                        },
                        // animated: true,
                    }
                    this.activities.push(linkEdge);
                }

            } else {
                if (data.expr.ast.type == 'select') {
                    this.selectAST(data.expr.ast, exLink,false);
                }
                // if (ast.type == 'insert') {
                //     insertAST(ast)
                // }
            }

        });


    });
}

function genEdge(ast, animated, exLink,linkAnimated) {
    if (ast.from.length > 1 || exLink != null) {
        let bf = '';
        for (let index = 0; index < ast.from.length; index++) {
            let data = ast.from[index];
            if (data.expr == undefined) {
                if (index == 0) {
                    bf = ((data.db != null) ? (data.db + '.') : '') + data.table;
                    if (exLink != null && bf!=exLink) {
                        const linkEdge = {
                            id: `${exLink}-${bf}`,
                            source: `${exLink}`,
                            target: `${bf}`,
                            style: (edge) => {
                                if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                                return { stroke: '#10b981', strokeWidth: 3 }
                            },
                            animated: `${linkAnimated}`,
                        }
                        this.activities.push(linkEdge);
                    }
                } else {
                    const cur = ((data.db != null) ? (data.db + '.') : '') + data.table;
                    const linkEdge = {
                        id: `${bf}-${cur}`,
                        source: `${bf}`,
                        target: `${cur}`,
                        animated: `${animated}`,
                        style: (edge) => {
                            if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                            return { stroke: '#10b981', strokeWidth: 3 }
                        },
                    }

                    this.activities.push(linkEdge);

                    if (exLink != null) {
                        const linkEdge = {
                            id: `${exLink}-${cur}`,
                            source: `${exLink}`,
                            target: `${cur}`,
                            style: (edge) => {
                                if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                                return { stroke: '#10b981', strokeWidth: 3 }
                            },
                            animated: `${linkAnimated}`,
                        }
                        this.activities.push(linkEdge);
                    }

                }
            } else {
                this.genEdge(data.expr.ast, exLink,linkAnimated);
            }

        }

    }
}

</script>