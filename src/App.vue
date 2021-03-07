<template>
  <div class="view">
    <div class="view__wrapper">
      
      <div class="selectable-fields">
        <div class="selectable-fields__wrapper">
          <div class="selectable-field">
            <select v-model="targetScope">
              <option 
                v-for="type in DOC_TYPES"
                :key="type"
                :value="type">
                {{ type }}
              </option>
            </select>

            <select v-model="targetFieldType">
              <option 
                v-for="fieldType in DOC_FIELD_TYPES"
                :key="fieldType"
                :value="fieldType">
                {{ fieldType }}
              </option>
            </select>
          </div>
          
          <template
            v-for="field in selectableFields"
            :key="field">
            <div class="selectable-field">
              <input
                :checked="selectedFields.includes(field)"
                @click.stop="selectedFields = selectedFields.includes(field) ? selectedFields.filter(_field => _field !== field) : [...selectedFields, field]"
                type="checkbox">
              <h5 :style="{'border-bottom': `solid ${ FIELD_COLORS[field] } 3px`}">{{ field }}</h5>
            </div>
          </template>
        </div>
      </div>

      <div class="graph">
        <div 
          ref="targetSvgContainer"
          class="graph__wrapper">
          <svg ref="targetSvg"></svg>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
/* eslint-disable no-unused-vars */
import ocr from '@/assets/ocr-stats-monhtly-from-2018.json'

import Moment from 'moment'
import * as d3 from 'd3'

import { 
  computed,
  onMounted,
  ref,
  watch
} from 'vue'

const DOC_TYPES = {
  DOCS_ALL: 'allDocuments',
  DOCS_INVOICES: 'byDocTypeInvoices',
  DOCS_RECEIPTS: 'byDocTypeReceipts',
}

const DOC_FIELD_TYPES = {
  CUMULATIVE: 'CumulativePrecision',
  REGOCNIGITION_FREQUENCY: 'RegocnitionFrequency',
  REGOCNIGITION_PRECISION: 'RegocnigitonPrecision',
}

const FIELDS = {
  bankAccounts: 'bankAccounts',
  currency: 'currency',
  documentId: 'documentId',
  dueDate: 'dueDate',
  invoiceReferenceNo: 'invoiceReferenceNo',
  issued: 'issued',
  summary: 'summary',
  supplier: 'supplier',
}


const FIELD_COLORS = {
  supplier: '#DF00F0',
  issued: '#EB0A00',
  summary: '#EBB800',
  currency: '#01EB08',
  dueDate: '#00E5EB',
  bankAccounts: '#DF00F0',
  documentId: '#2B05FB',
  invoiceReferenceNo: '#FFFFFF',
}

export default {
  setup () {
    const targetSvgContainer = ref(null)
    const targetSvg = ref(null)
    const targetScope = ref(DOC_TYPES.DOCS_ALL)
    const selectableFields = ref(Object.keys(FIELDS))
    const selectedFields = ref(selectableFields.value)
    const targetFieldType = ref(DOC_FIELD_TYPES.CUMULATIVE)

    const defaultMargin = 40

    const sizes = computed(() => {  
      return {
        width: targetSvg.value.getBoundingClientRect().width,
        height: targetSvg.value.getBoundingClientRect().height,
        margin: {
          top: defaultMargin, 
          right: defaultMargin, 
          bottom: defaultMargin, 
          left: defaultMargin
        },
      }
    })
    
    const timeline = computed(() => {
      return Object
        .entries(ocr)
        .reduce((accumulator, [monthOfaYear, statistics], index, final) => {
          return [
            ...accumulator,
            [new Moment(monthOfaYear, 'MM.YYYY').clone().toDate(), statistics[targetScope.value]?.Fields]
          ]
        }, [])
    })

    const overallMonths = computed(() => {
      return timeline.value
        .map(([monthOfaYear]) => monthOfaYear)
    })

    const allPossibleFields = computed(() => {
      return selectedFields.value
      // return [
      //   'bankAccounts',
      //   'currency',
      //   'documentId',
      //   'dueDate',
      //   'invoiceReferenceNo',
      //   'issued',
      //   'summary',
      //   'supplier',
      // ]
    })

    const yScale = computed(() => {
      return d3
        .scaleLinear()
        .range([sizes.value.height - ((sizes.value.margin.top + sizes.value.margin.bottom) * 2), 0])
        .domain([0, 100])
    })

    const xScale = computed(() => {
      return d3
        .scaleTime()
        .range([0, sizes.value.width - ((sizes.value.margin.left + sizes.value.margin.right) * 2)])
        .domain(d3.extent(overallMonths.value))
    })

    function renderSvg (target) {
      d3
        .select(target)
        .selectAll('*')
        .remove()

      const graph = d3
        .select(target)
        .attr('width', sizes.value.width - sizes.value.margin.left)
        .attr('height', sizes.value.height)
          .append('g')
          .attr('transform', `translate(${ sizes.value.margin.left }, ${ 0 })`)

      renderGrid(graph)
      renderYAxis(graph)
      renderXAxis(graph)
      renderTimeline(graph)
      renderToolTip(graph)

      console.log({timeline: timeline.value, overallMonths: overallMonths.value, graph, sizes: sizes.value, allPossibleFields: allPossibleFields.value})
    }

    function renderYAxis (_graph) {
      _graph
        .append('g')
        .attr('transform', `translate(${ sizes.value.margin.left }, ${ sizes.value.margin.top + sizes.value.margin.bottom })`)
        .attr('class', 'axis axis--y')
        .attr('color', 'white')
        .call(d3.axisLeft(yScale.value))
    }

    function renderXAxis (_graph) {
      const xAxis = d3
        .axisBottom(xScale.value)
        .ticks(30.4167)
        .tickFormat((a) => {
          if (Moment(a).startOf('year').isSame(Moment(a))) {
            return d3.timeFormat("%Y")(a)
          }

          return d3.timeFormat("%b")(a)
        })
      
      _graph
        .append('g')
        .attr('transform', `translate(${ sizes.value.margin.left }, ${ sizes.value.height - (sizes.value.margin.bottom * 2) })`)
        .attr('class', 'axis axis--x')
        .attr('color', 'white')
        .call(xAxis)
    }

    function calcLine (_timeline, _field) {
      const _line =  d3
        .line()
        // .curve(d3.curveBasis)
        .curve(d3.curveMonotoneX)
        // .curve(d3.curveStepAfter)
        .x(([monthOfaYear, _]) => xScale.value(monthOfaYear))
        .y(([_, fields]) => {
          if (fields[_field]?.VerifiedCount < 20) {
            return sizes.value.height - (sizes.value.margin.top * 4)
          }

          return yScale.value(fields[_field]?.[targetFieldType.value]) ?? 0
        })

      return _line(_timeline)
    }

    function renderTimeline (_graph) {
      const tooltip = renderToolTip()

      return allPossibleFields.value
        .forEach((field) => {
          _graph
            .append('path')
            .attr('class', `line line--${ field }`)
            .attr('d', calcLine(timeline.value, field))
            .on('mouseover', ({currentTarget}) => {
              d3
                .select(currentTarget)
                .style('stroke-width', 4)

              d3
                .selectAll(`.line:not(.line--${ field })`)
                .style('opacity', '.2')

              d3
                .selectAll(`.dot:not(.dot--${ field })`)
                .style('opacity', '.2')
            })
            .on('mouseout', ({currentTarget}) => {
              d3
                .select(currentTarget)
                .style('stroke-width', 2)

              d3
                .selectAll(`.dot--${ field }`)
                .style('stroke', FIELD_COLORS[field])

              d3
                .selectAll(`.line:not(.line--${ field })`)
                .style('opacity', '1')

              d3
                .selectAll(`.dot:not(.dot--${ field })`)
                .style('opacity', '1')
            })
            .attr('stroke-dasharray', `${ 4000 } ${ 4000 }`)
            .attr('stroke-dashoffset', 4000)
            .attr('transform', `translate(${ sizes.value.margin.left }, ${ sizes.value.margin.top * 2 })`)
            .style('fill', 'none')
            .style('stroke', FIELD_COLORS[field])
            .style('stroke-width', 4)
            .transition()
            .ease(d3.easeLinear, 8)
            .duration(2500)
            .styleTween('stroke', () => d3.interpolateRgb('white', FIELD_COLORS[field]))
            .style('stroke-width', 2)
            .attr('stroke-dashoffset', 0)

          renderDots(_graph, field, tooltip)
        })
        
    }

    function renderDots(_graph, _field, _tooltip) {
      _graph
        .selectAll('dot')
        .data(timeline.value)
        .enter()
        .append('circle')
        .on('mouseover', ({currentTarget} = {}) => {
          const {dataset} = currentTarget
          const date = new Moment(dataset.date).format('MMMM YYYY')
          const monthOfaYear = new Moment(dataset.date).format('MM.YYYY')

          let x = xScale.value(new Moment(dataset.date)) + (sizes.value.margin.left * 2) + 20
          let y = yScale.value(ocr[monthOfaYear]?.[targetScope.value]?.Fields[_field]?.[targetFieldType.value] ?? 0) + (sizes.value.margin.top * 2) + 20

          if (x > (sizes.value.width / 2)) {
            x = x - 420
          }

          if (y > (sizes.value.height / 2)) {
            y = y - 330
          }
          
          _tooltip
            .style('opacity', 1)
            .style('left', `${ x }px`)
            .style('top', `${ y }px`)
            .style('display', 'unset')
            .html(`
              <div class="tooltip__wrapper">
                <h1>${ _field }</h1>
                <h5>${ date }</h5>
                <br>
                ${ Object.entries(ocr[monthOfaYear]?.[targetScope.value]?.Fields[_field]).map(([key, value]) => `
                  <div class="data">
                    <h5><i>${ key }</i></h5>
                    <h5>${ value }</h5>
                  </div>
                `).join('') }
              </div>
            `)
          
          d3
            .select(currentTarget)
            .attr('r', 8)
            .style('fill', 'white')
            .style('stroke', 'white')

          d3
            .selectAll(`.line--${ _field }`)
            .style('stroke-width', 4)
          
          d3
            .selectAll(`.line:not(.line--${ _field })`)
            .style('opacity', '.2')

          d3
            .selectAll(`.dot:not(.dot--${ _field })`)
            .style('opacity', '.2')
        })
        .on('mouseout', ({currentTarget}) => {
          d3
            .select(currentTarget)
            .attr('r', 4)
            .style('stroke', FIELD_COLORS[_field])
            .style('fill', '#151515')

          _tooltip
            .style('opacity', 0)
            .style('display', 'none')

          d3
            .selectAll(`.line--${ _field }`)
            .style('stroke-width', 2)
            .style('stroke', FIELD_COLORS[_field])

          d3
            .selectAll(`.line:not(.line--${ _field })`)
            .style('opacity', '1')

          d3
            .selectAll(`.dot:not(.dot--${ _field })`)
            .style('opacity', '1')
        })
        .attr('data-date', ([monthOfaYear]) => monthOfaYear)
        .attr('data-field', _field)
        .attr('class', `dot dot--${ _field }`)
        .attr('transform', `translate(${ sizes.value.margin.left }, ${ sizes.value.margin.top * 2 })`)
        .attr('cx', ([monthOfaYear, _]) => xScale.value(monthOfaYear))
        .attr('cy', ([_, fields]) => {
          if (fields[_field]?.VerifiedCount < 10) {
            return sizes.value.height - (sizes.value.margin.top * 4)
          }

          return yScale.value(fields[_field]?.[targetFieldType.value]) ?? 0
        })
        .transition()
        .delay(2500)
        .ease(d3.easeLinear, 8)
        .duration(100)
        .style('stroke-width', 2)
        .style('fill', '#151515')
        .attr('r', 4)
        .styleTween('stroke', () => d3.interpolateRgb('white', FIELD_COLORS[_field]))
        // .attr('fill', FIELD_COLORS[_field])
    }

    function renderGrid (_graph) {
      const xGridLines = d3
        .axisBottom(xScale.value)
        .ticks(30.4167)
        .tickSize(sizes.value.height - (sizes.value.margin.bottom * 4))
        .tickFormat('')

      _graph
        .style('color', '#ffffff10')
        .append('g')
        .style('stroke-width', 1)
        .attr('class', 'grid')
        .attr('transform', `translate(${ sizes.value.margin.left }, ${ sizes.value.margin.top * 2 })`)
        .call(xGridLines)

      const yGridLines = d3
        .axisLeft(yScale.value)
        .ticks(10)
        .tickSize(sizes.value.width - (sizes.value.margin.left * 4))
        .tickFormat('')

      _graph
        .style('color', '#ffffff10')
        .append('g')
        .style('stroke-width', 1)
        .attr('class', 'grid')
        .attr('transform', `translate(${ sizes.value.width - (sizes.value.margin.left * 3) }, ${ sizes.value.margin.top * 2 })`)
        .call(yGridLines)
    }

    function renderToolTip (_graph) {
      return d3
        .select(targetSvgContainer.value)
        .append('div')
        .attr('class', 'tooltip')
        .style('opacity', 0)
        .style('display', 'none')
        .style('position', 'absolute')
        .attr('pointer-events', 'none')
        .attr('user-select', 'none')
    }


    function selectTarget (target) {
      targetScope.value = target
    }

    watch(() => [
      targetScope.value,
      selectedFields.value,
      targetFieldType.value,
    ], () => {
      renderSvg(targetSvg.value)
    })
 
    onMounted(() => {
      renderSvg(targetSvg.value)
    })

    return {
      targetSvgContainer,
      allPossibleFields,
      selectableFields,
      DOC_FIELD_TYPES,
      targetFieldType,
      selectedFields,
      selectTarget,
      FIELD_COLORS,
      targetScope,
      targetSvg,
      DOC_TYPES,
    }
  }
}
</script>

<style lang="scss">
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@600&family=Poppins:ital,wght@0,500;0,600;0,700;1,500&display=swap');

body {
  font-family: Poppins, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #e5e5e5;
  margin: 0;
  padding: 0;
}

*,*::after,*::before {
  box-sizing: border-box;
}

p,h1,h2,h3,h4,h5,h6,span {
  margin: 0;
}

.view {
  &__wrapper {
    min-height: 100vh;
    background-color: #313131;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    padding-bottom: 10vh;
  }
}

.graph {
  &__wrapper {
    width: 94vw;
    height: 90vh;
    background-color: #00000090;
    border-radius: 6px;
    box-shadow: 0 22px 35px -19px #00000060;
    position: relative;

    svg {
      width: 100%;
      height: 100%;
    }
  }
}

.selectable-fields {
  display: flex;
  flex-grow: 1;
  flex-shrink: 0;
  padding: 20px 0 ;

  &__wrapper {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
}

.selectable-field {
  display: flex;
  align-items: center;
  padding: 8px 12px;

  h5 {
    color: white;
    margin: 0 8px;
    font-weight: 500;
  }
}

.tooltip {
  &__wrapper {
    background-color: #ffffff;
    padding: 40px;
    color: #313131;

    h1 {
      text-transform: uppercase;
    }

    .data {
      margin-top: 10px;
      min-width: 300px;
      display: flex;
      justify-content: space-between;
    }
  }
}

.domain {
  stroke: #FFFFFF10;
}

.tick > line {
  stroke: #FFFFFF10;
}
</style>
