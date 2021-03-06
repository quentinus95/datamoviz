<template>
  <section class="map" data-aos="fade">
    <div class="container">
      <div class="row">
        <div class="col-12">
          <h3 class="get-started">Production<br />countries</h3>
          <div ref="map" class="map"></div>
          <div class="row">
            <div class="col-12 col-md-4">
              <small>
                <span class="badge badge-info">Pro tip</span>
                Click on several countries to get movies produced there. Click again to unselect.
              </small>
            </div>
            <div class="col-12 col-md-4 arrow-down">
              <a href="#" class="go-down"><i class="fa fa-arrow-circle-down"></i></a>
            </div>
            <div class="col-12 col-md-4">
              <template v-if="paletteScale">
                <span v-for="b, key in paletteScale.clusters()" :key="key" class="legend">
                  <span :style="{ background: paletteScale.range()[key] }" class="color"></span>
                  <span class="limit">{{ b }}</span>
                </span>
                <span class="legend">
                  <span :style="{ background: paletteScale.range()[paletteScale.clusters().length] }" class="color"></span>
                </span>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
  import * as d3 from 'd3';
  import Elevator from 'elevator.js';
  import Datamaps from 'datamaps';
  import IsoConverter from 'i18n-iso-countries';
  import { FILTERS_UPDATE } from '../event-bus';
  import scaleCluster from 'd3-scale-cluster';

  let map;

  export default {
    name: 'app-map',
    data() {
      return {
        countries: [],
        allCountries: {},
        paletteScale: null,
        elevator: null
      };
    },
    methods: {
      countCountries(filters) {
        filters = filters || {};

        return fetch(`${process.env.SERVER_URL}/count/countries?filters=${encodeURI(JSON.stringify(filters))}`)
          .then(response => response.json())
          .then((json) => {
            this.countries = this.formatCountries(json);

            if (Object.keys(this.allCountries).length === 0) {
              this.countries.forEach((item) => {
                this.allCountries[item.iso] = { value: 0, country: item._id, fillColor: '#3f4045' };
              });
            }

            this.displayCountries();
          });
      },
      formatCountries(countries) {
        return countries.map((item) => {
          return {
            iso: this.translateCountryCode(item._id.iso_3166_1),
            original: item
          }
        });
      },
      displayCountries() {
        map.updateChoropleth(this.allCountries);

        if (this.countries.length === 0) {
          return;
        }

        this.paletteScale = scaleCluster() // Natural breaks
          .domain(this.countries.map(c => c.original.value))
          .range(['#dee7f3','#becee6','#9db8da','#7aa1cd','#548bc1','#1975b4']);

        const dataMapCountries = {};
        this.countries.forEach((item) => {
          dataMapCountries[item.iso] = { country: item.original._id, value: item.original.value, fillColor: this.paletteScale(item.original.value) };
        });

        map.updateChoropleth(dataMapCountries);
      },
      loadMap() {
        map = new Datamaps({
          element: this.$refs.map,
          projection: 'mercator',
          fills: { defaultFill: '#3f4045' },
          data: {},
          responsive: true,
          geographyConfig: {
            borderColor: '#333439',
            popupOnHover: true,
            highlightBorderWidth: 2,
            highlightFillColor(geo) {
              return geo.fillColor || '#3f4045';
            },
            highlightBorderColor: '#505156',
            popupTemplate(geo, data) {
              if (!data) {
                return;
              }
              return `<div class="hoverinfo">${data.country.name}: ${data.value} movies</div>`;
            }
          },
          done: (datamap) => {
            datamap.svg.selectAll('.datamaps-subunit').on('click', (geography) => {
              this.filterCountry(map.options.data[geography.id].country)
            });
          }
        });
        window.addEventListener('resize', () => { map.resize(); });
      },
      translateCountryCode (code) {
        const oldCountries = {SU: 'RU', XG: 'DE', CS: 'RS', YU: 'RS', XC: 'SK', AN: 'NL'};

        if (oldCountries[code]) {
          code = oldCountries[code];
        }

        return IsoConverter.alpha2ToAlpha3(code);
      },
      filterCountry (country) {
        const { filters } = this.$bus;

        if (!filters.hasOwnProperty('production_countries')) {
          filters.production_countries = { $all: [] }
        }

        const found = filters.production_countries.$all.find(c => c.iso_3166_1 === country.iso_3166_1);
        if (found !== undefined) {
          filters.production_countries.$all = filters.production_countries.$all.filter(c => c.iso_3166_1 !== found.iso_3166_1);

          if (filters.production_countries.$all.length === 0) {
            delete filters.production_countries;
          }
        } else {
          filters.production_countries.$all.push(country);
        }

        this.$bus.$emit(FILTERS_UPDATE, filters);
      }
    },
    mounted() {
      this.loadMap();
      this.countCountries();
      this.$bus.$on(FILTERS_UPDATE, (filters) => {
        this.countCountries(filters);
      });
      this.elevator = new Elevator({
        element: document.querySelector('.go-down'),
        targetElement: document.querySelector('.go-down'),
        duration: 1000
      });
    }
  };

</script>

<style lang="scss" ref="stylesheet/scss">
  @import '../scss/vars';

  section.map {
    padding: 0 0 10px;

    h3.get-started {
      display: none;
      position: absolute;
      top: 25px;
      left: 15px;
      line-height: 1.2em;
      z-index: 1000;
      text-shadow: -2px 0 $global-color-background, 0 2px $global-color-background, 2px 0 $global-color-background, 0 -2px $global-color-background;

      @media screen and (min-width: 500px) {
        display: inline;
      }
    }

    div.map {
      margin-bottom: 10px;

      svg path {
        cursor: pointer;
      }
    }

    .legend {
      font-size: 0.8em;
      float: right;
      position: relative;

      .color {
        width: 50px;
        height: 10px;
        display: inline-block;
      }

      .limit {
        position: absolute;
        bottom: -11px;
        left: -25px;
        font-size: 0.75em;
        width: 50px;
        text-align: center;
      }
    }

    small {
      display: inline-block;
      line-height: 1.4em;
    }

    div.arrow-down {
      text-align: center;

      a {
        color: white;

        .fa {
          font-size: 2em;
        }
      }
    }
  }

  .datamaps-hoverover .hoverinfo {
    color: $global-color-background;
    box-shadow: 7px 7px 12px -9px #777;
    border: 1px solid #CCC;
  }
</style>
