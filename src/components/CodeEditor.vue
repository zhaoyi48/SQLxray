<template>
    <el-row>
        <el-col :span="12">
            <codemirror v-model="code" placeholder="Code goes here..." :style="{ height: '90vh' }" :autofocus="true"
                :indent-with-tab="true" :tab-size="4" :extensions="extensions" @ready="handleReady"
                @change="flowgen($event)" />
        </el-col>
        <el-col :span="12">
            <SqlFlow />
        </el-col>
    </el-row>
</template>

<script setup>
import { ref, shallowRef, provide, } from 'vue'
import { Codemirror } from 'vue-codemirror'
import { sql } from '@codemirror/lang-sql'
import { oneDark } from '@codemirror/theme-one-dark'
import SqlFlow from './SqlFlow.vue'
import { Parser } from 'node-sql-parser'

const activities = ref([])
provide('map', activities)

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

function flowgen(sqls) {
    this.activities.length = 0;
    var i = 1;
    const infoTree = {
    }
    sqls.split(';').forEach(sql => {
        if (sql.trim() != '') {
            try {
                const parser = new Parser();
                const ast = parser.astify(sql);
                console.log('change', ast);
                const output = {};
                var tableInfo = '';
                const tableCount = ast.from.length;
                var bfTable = '';
                var j = 0;
                var bias=0;
                ast.from.forEach(data => {
                    const tablename = ((data.db != null) ? (data.db + '.') : '') + data.table;
                    const newNode = {
                        id: tablename,
                        label: `${tablename}`,
                        position: { },
                    }
                    if(infoTree[`${tablename}`]){
                        bias++;
                    }
                    if (j == 0) {
                        bfTable = tablename;
                        newNode.position={ x: 100, y: i * 100 }
                        if(!infoTree[`${tablename}`]){
                            this.activities.push(newNode);
                            infoTree[`${tablename}`]={}
                        }
                    } else {
                        const linkEdge = {
                            id: `${bfTable}-${tablename}`,
                            source: `${bfTable}`,
                            target: `${tablename}`,
                        }
                        newNode.position={ x: 100, y: (i+bias) * 100 }
                        bfTable = tablename;
                        this.activities.push(linkEdge);
                        this.activities.push(newNode);
                    }
                    i++;
                    j++;
                    // tableInfo = tableInfo + (tableInfo == '' ? '' : ' ') + tablename;


                });
                // var columnsInfo = '';
                // if (typeof (ast.columns) == "string") {
                //     columnsInfo = ast.columns
                // } else {
                //     ast.columns.forEach(colData => {
                //         const columnsName = ((colData.expr.table != null) ? (colData.expr.table + '.') : '') + colData.expr.column;
                //         columnsInfo = columnsInfo + (columnsInfo == '' ? '' : ',') + columnsName;
                //     });
                // }
                // output['columns'] = columnsInfo;
                output['table'] = tableInfo;
                // console.log('change', output);
            } catch (error) {
                this.activities.length = 0;
                console.error(error);
            }
            console.log('change', this.activities);
        }
    });

}
</script>