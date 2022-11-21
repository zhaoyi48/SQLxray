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
import { Parser } from '@florajs/sql-parser'

let activities = ref([]);
provide('map', activities);
let infoTree = {};
let yBias = 1;
let tmpCounter = 1;

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
    infoTree = {};
    yBias = 1;
    activities.value.length = 0;
    sqls.split(';').forEach(sql => {
        if (sql.trim() != '') {
            try {
                const parser = new Parser();
                const ast = parser.parse(sql);
                console.log('change', ast);

                if (ast.type == 'select') {
                    selectAST(ast, null, true)
                }

                if (ast.type == 'insert') {
                    insertAST(ast)
                }


            } catch (error) {
                activities.value.length = 0;
                console.error(error);
            }
            console.log('change', activities.value);
        }
    });

}

function selectAST(ast, bfAst) {
    var maxXbias = 0;
    ast.from.forEach(data => {
        const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        if (infoTree[`${tablename}`]) {
            if (maxXbias < infoTree[`${tablename}`].xBias) {
                maxXbias = infoTree[`${tablename}`].xBias;
            }
        }
    });
    ast.from.forEach(data => {
        if (data.expr == undefined) {
            const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
            if (!infoTree[`${tablename}`]) {
                infoTree[`${tablename}`] = {
                    xBias: `${++maxXbias}`,
                    yBias: `${yBias}`,
                    sourceTable: []
                }
                const newNode = {
                    id: tablename,
                    label: `${tablename}`,
                    position: { x: 0, y: 0 },
                }
                newNode.position = { x: infoTree[`${tablename}`].xBias * 100, y: infoTree[`${tablename}`].yBias * 100 }
                activities.value.push(newNode);
                yBias++;
            }
        } else {
            if (data.expr.ast.type == 'select') {
                selectAST(data.expr.ast, ast);
            }

        }

    });
    genEdge(ast, bfAst, null);

}

function insertAST(ast) {
    var maxXbias = 0;
    ast.table.forEach(data => {
        const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        if (infoTree[`${tablename}`]) {
            if (maxXbias < infoTree[`${tablename}`].xBias) {
                maxXbias = infoTree[`${tablename}`].xBias;
            }
        }
    });

    ast.table.forEach(data => {
        const baseTablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        if (!infoTree[`${baseTablename}`]) {
            infoTree[`${baseTablename}`] = {
                xBias: `${++maxXbias}`,
                yBias: `${yBias}`,
                sourceTable: []
            }
            const newNode = {
                id: baseTablename,
                label: `${baseTablename}`,
                position: { x: 0, y: 0 },
            }
            newNode.position = { x: infoTree[`${baseTablename}`].xBias * 100, y: infoTree[`${baseTablename}`].yBias * 100 }
            activities.value.push(newNode);
            yBias++;
        }

        selectAST(ast.values, ast);

        // ast.values.from.forEach(data => {
        //     if (data.expr == undefined) {

        //         const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
        //         if (!infoTree[`${tablename}`]) {
        //             infoTree[`${tablename}`] = {
        //                 xBias: `${++maxXbias}`,
        //                 yBias: `${yBias}`,
        //                 sourceTable: []
        //             }
        //             const newNode = {
        //                 id: tablename,
        //                 label: `${tablename}`,
        //                 position: { x: 0, y: 0 },
        //             }
        //             newNode.position = { x: infoTree[`${tablename}`].xBias * 100, y: infoTree[`${tablename}`].yBias * 100 }
        //             activities.value.push(newNode);
        //             infoTree[`${baseTablename}`].sourceTable.push(tablename);
        //             const linkEdge = {
        //                 id: `${baseTablename}-${tablename}`,
        //                 source: `${baseTablename}`,
        //                 target: `${tablename}`,
        //                 style: (edge) => {
        //                     if (!edge.sourceNode.selected && !edge.targetNode.selected) return
        //                     return { stroke: '#10b981', strokeWidth: 3 }
        //                 },
        //                 // animated: true,
        //             }
        //             activities.value.push(linkEdge);
        //             yBias++;
        //         } else {
        //             const linkEdge = {
        //                 id: `${baseTablename}-${tablename}`,
        //                 source: `${baseTablename}`,
        //                 target: `${tablename}`,
        //                 style: (edge) => {
        //                     if (!edge.sourceNode.selected && !edge.targetNode.selected) return
        //                     return { stroke: '#10b981', strokeWidth: 3 }
        //                 },
        //                 // animated: true,
        //             }
        //             activities.value.push(linkEdge);
        //         }

        //     } else {
        //         if (data.expr.ast.type == 'select') {
        //             selectAST(data.expr.ast,ast,false);
        //         }
        //         // if (ast.type == 'insert') {
        //         //     insertAST(ast)
        //         // }
        //     }

        // });


    });
}

function genEdge(ast, bfAst, bfnode) {
    if (bfAst != null && bfAst.type == 'insert') {
        bfAst.table.forEach(data => {
            const baseTablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
            if (ast.from.length > 0) {
                let bf = '';
                for (let index = 0; index < ast.from.length; index++) {
                    let data = ast.from[index];
                    if (data.expr == undefined) {
                        if (index == 0) {
                            bf = ((data.db != null) ? (data.db + '.') : '') + data.table;
                            const linkEdge = {
                                id: `${baseTablename}-${bf}`,
                                source: `${baseTablename}`,
                                target: `${bf}`,
                                style: (edge) => {
                                    if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                                    return { stroke: '#10b981', strokeWidth: 3 }
                                },
                                animated: false,
                            }
                            activities.value.push(linkEdge);
                        } else {
                            const cur = ((data.db != null) ? (data.db + '.') : '') + data.table;
                            const linkEdge = {
                                id: `${bf}-${cur}`,
                                source: `${bf}`,
                                target: `${cur}`,
                                animated: true,
                                style: (edge) => {
                                    if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                                    return { stroke: '#10b981', strokeWidth: 3 }
                                },
                            }

                            activities.value.push(linkEdge);

                            const exlinkEdge = {
                                id: `${baseTablename}-${cur}`,
                                source: `${baseTablename}`,
                                target: `${cur}`,
                                style: (edge) => {
                                    if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                                    return { stroke: '#10b981', strokeWidth: 3 }
                                },
                                animated: false,
                            }
                            activities.value.push(exlinkEdge);

                        }
                    } else {
                        genEdge(data.expr.ast, bfAst, null);
                    }

                }

            }
        });

    } else {
        if (ast.from.length > 0) {
            let bf = bfnode != null ? bfnode : '';
            for (let index = 0; index < ast.from.length; index++) {
                let data = ast.from[index];
                if (data.expr == undefined) {
                    if (index == 0 && bf == '') {
                        bf = ((data.db != null) ? (data.db + '.') : '') + data.table;
                    } else {
                        const cur = ((data.db != null) ? (data.db + '.') : '') + data.table;
                        const linkEdge = {
                            id: `${bf}-${cur}`,
                            source: `${bf}`,
                            target: `${cur}`,
                            animated: true,
                            style: (edge) => {
                                if (!edge.sourceNode.selected && !edge.targetNode.selected) return
                                return { stroke: '#10b981', strokeWidth: 3 }
                            },
                        }

                        activities.value.push(linkEdge);

                    }
                } else {
                    if (index == 0) {
                        //create tmp-table node
                        const tablename='AUTOGEN_tmp'+tmpCounter;
                        infoTree[`${tablename}`] = {
                            xBias: 0,
                            yBias: `${yBias}`,
                            sourceTable: []
                        }
                        const newNode = {
                            id: tablename,
                            label: `${tablename}`,
                            position: { x: 0, y: 0 },
                        }
                        newNode.position = { x: infoTree[`${tablename}`].xBias * 100, y: infoTree[`${tablename}`].yBias * 100 }
                        activities.value.push(newNode);
                        yBias++;
                        tmpCounter++;
                        bf=tablename;
                    }
                    genEdge(data.expr.ast, bfAst, bf);
                }

            }

        }
    }
}

</script>