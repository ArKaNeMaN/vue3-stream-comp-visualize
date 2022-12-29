<script setup>
import { onMounted, ref } from 'vue';
import { cloneDeep } from 'lodash';

function deepCopyObj(obj) {
  return cloneDeep(obj);
}

function makeIfElse(outIf, outElse) {
  return {
    func(ins) {
      return callIfHas(ins, (ins) => {
        if (ins.cond) {
          return { outIf: ins.value };
        } else {
          return { outElse: ins.value };
        }
      }, ['value', 'cond']);
    },
    ins: {
      value: null,
      cond: null,
    },
    outs: { outIf: outIf, outElse: outElse },
  };
}

function makeTrigger(value, outTo) {
  return {
    func(ins) {
      return callIfHas(ins, () => {
        return { out: value };
      }, ['in']);
    },
    ins: {
      in: null,
    },
    outs: { out: outTo },
  };
}

function makeMerg(outTo) {
  return {
    func(ins) {
      if (ins.i1 !== null) {
        return { out: ins.i1 };
      }

      if (ins.i2 !== null) {
        return { out: ins.i2 };
      }
      
      return null;
    },
    ins: {
      i1: null,
      i2: null,
    },
    outs: { out: outTo },
  };
}

function makeSplit(out1, out2) {
  return {
    func(ins) {
      return callIfHas(ins, (ins) => {
        return {
          o1: ins.in,
          o2: ins.in,
        };
      }, ['in']);
    },
    ins: {
      in: null,
    },
    outs: { o1: out1, o2: out2 },
  };
}

function makeSimpleInput(outTo, value) {
  return {
    func(ins) {
      return callIfHas(ins, (ins) => {
        return { out: ins.value };
      }, ['value']);
    },
    ins: {
      value: value,
    },
    outs: {
      out: outTo,
    },
  };
}

function makeValve(outTo) {
  return {
    func(ins) {
      return callIfHas(ins, (ins) => {
        if (ins.cond) {
          return { out: ins.value };
        } else {
          return null;
        }
      }, ['value', 'cond']);
    },
    ins: {
      value: null,
      cond: null,
    },
    outs: { out: outTo },
  };
}

function callIfHas(ins, cb, reqIns) {
  for (let inName of reqIns) {
    if (ins[inName] === null) {
      return null;
    }
  }
  return cb(ins);
}

const inputArray = [87, 54, 21, 87, 55, 666];

const table = {
  inArr0: makeSimpleInput('mergMin.i1', inputArray[0]),
  inArr: {
    func(ins) {
      if (!ins.arr.length) {
        return null;
      }

      return { out: ins.arr.pop() };
    },
    manualIns: true,
    ins: {
      arr: inputArray,
    },
    outs: {
      out: 'min.n2',
    },
  },
  inLen: makeSimpleInput('mergLen.i1', inputArray.length),

  mergMin: makeMerg('min.n1'),
  
  min: {
    func(ins) {
      return callIfHas(ins, (ins) => {
        return { out: Math.min(ins.n1, ins.n2) };
      }, ['n1', 'n2']);
    },
    ins: {
      n1: null,
      n2: null,
    },
    outs: { out: 'splitMin.in' },
  },

  dec: {
    func(ins) {
      return callIfHas(ins, (ins) => {
        return { out: ins.value - 1 };
      }, ['value']);
    },
    ins: {
      value: null,
    },
    outs: { out: 'lenDecValve.value' },
  },

  splitMin: makeSplit('resIfElse.value', 'valveTrigger.in'),

  resIfElse: makeIfElse('mergMin.i2', 'out.res'),

  mergLen: makeMerg('dec.value'),
  valveTrigger: makeTrigger(true, 'lenDecValve.cond'),
  lenDecValve: makeValve('splitDec.in'),
  splitDec: makeSplit('resIfElse.cond', 'mergLen.i2'),

  // Служебный блок
  out: {
    func() {
      return null;
    },
    ins: { res: null },
    outs: {},
  }
};

function streamStep(table) {
  let newTable = deepCopyObj(table);

  Object.keys(table).forEach((nodeName) => {
    let ins = cloneDeep(table[nodeName].ins);
    let outs = table[nodeName].func(ins);
    if (outs === null) {
      return;
    }

    let outBusy = false;
    Object.keys(table[nodeName].outs).forEach((outName) => {
      const out = table[nodeName].outs[outName].split('.');
      if (newTable[out[0]].ins[out[1]] === null) {
        newTable[out[0]].ins[out[1]] = outs[outName] ?? null;
      } else {
        outBusy = true;
      }
    });
    if (outBusy) {
      return;
    }

    if (table[nodeName].manualIns ?? false) {
      newTable[nodeName].ins = ins;
      return;
    }

    Object.keys(table[nodeName].ins).forEach((inName) => {
      newTable[nodeName].ins[inName] = null;
    });
  });

  return newTable;
}

function streamProc(table) {
  while (table.out.ins.res === null) {
    table = streamStep(table);
  }

  return table;
}

function tableToTable(table) {
  let res = {};
  // console.log(table);
  Object.keys(table).forEach((nodeName) => {
    let node = {};

    let outs = [];
    // console.log(table[nodeName]);
    let outVals = table[nodeName].func(cloneDeep(table[nodeName].ins));
    // console.log(outVals);

    Object.keys(table[nodeName].outs).forEach((outName) => {
      let out = table[nodeName].outs[outName];
      if (outVals !== null) {
        let val = outVals[outName] ?? null;
        if (['Array', 'Object'].includes(typeof(val))) {
          val = JSON.stringify(val);
        }
        outs.push(`${out} (${val})`);
      } else {
        outs.push(`${out} (-)`);
      }
    });
    node.outs = outs.join(', ');

    let ins = [];
    Object.keys(table[nodeName].ins).forEach((inName) => {
      ins.push(`${inName} (${table[nodeName].ins[inName] ?? '-'})`);
    });
    node.ins = ins.join(', ');

    res[nodeName] = node;
  });
  return res;
}

const refTable = ref(null);

onMounted(async () => {
  refTable.value = deepCopyObj(table);
  // console.log(await streamProc(refTable.value));
});

function onStep() {
  refTable.value = streamStep(refTable.value);
}

function onProc() {
  refTable.value = streamProc(refTable.value);
}

function onReset() {
  refTable.value = cloneDeep(table);
}

</script>

<template>
  <div class="p-8">
    <table v-if="refTable">
      <thead>
        <tr class="text-left border-b-2 border-gray-500/20">
          <th class="px-4">Нода</th>
          <th class="px-4">Входы</th>
          <th class="px-4">Выходы</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="(data, nodeName) in tableToTable(refTable)"
          :key="nodeName"
          class="border-b border-gray-500/20"
        >
          <td class="px-4">{{ nodeName }}</td>
          <td class="px-4">{{ data.ins }}</td>
          <td class="px-4">{{ data.outs }}</td>
        </tr>
      </tbody>
    </table>
    
    <div class="py-6 space-x-2">
      <button @click="onStep" class="px-2 py-0.5 border border-gray-400 rounded-lg">Шаг</button>
      <button @click="onProc" class="px-2 py-0.5 border border-gray-400 rounded-lg">До конца</button>
      <button @click="onReset" class="px-2 py-0.5 border border-gray-400 rounded-lg">Сбросить</button>
    </div>
  </div>
</template>
