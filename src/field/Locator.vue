<template>
    <k-field :input="_uid" v-bind="$props" :class="['k-locator-field', {filled: valueExists}, status]">
        <!-- Edit button -->
        <template slot="options">
            <k-button v-if="valueExists && filledStatus == 'closed'" :id="_uid" icon="edit" @click="toggle('open')">
                {{ $t('edit') }}
            </k-button>
            <k-button v-if="valueExists && filledStatus == 'open'" :id="_uid" icon="trash" @click="resetValue">
                {{ $t('locator.reset') }}
            </k-button>
            <k-button v-if="valueExists && filledStatus == 'open'" :id="_uid" icon="collapse" @click="toggle('closed')">
                {{ $t('locator.collapse') }}
            </k-button>
        </template>
        <div class="k-input k-locator-input" data-theme="field">
            <input ref="input" v-model="location" class="k-text-input" :placeholder="$t('locator.placeholder')" @input="onLocationInput">
            <button :class="[{disabled: !location.length}]" @click="getCoordinates"><svg><use href="#icon-locator-locate" /></svg> {{ $t('locator.locate') }}</button>
            <k-dropdown-content v-if="autocomplete" ref="dropdown">
                <k-dropdown-item v-for="(option, index) in dropdownOptions"
                                 :key="index"
                                 @click="select(option)"
                                 @keydown.native.enter.prevent="select(option)"
                                 @keydown.native.space.prevent="select(option)">
                    <span v-html="option.name" />
                    <span class="k-location-type" v-html="option.type" />
                </k-dropdown-item>
            </k-dropdown-content>
        </div>
        <k-dialog ref="dialog" @close="error = ''">
            <k-text>{{ error }}</k-text>
            <k-button-group slot="footer">
                <k-button icon="check" @click="$refs.dialog.close()">
                    {{ $t("confirm") }}
                </k-button>
            </k-button-group>
        </k-dialog>

        <div class="k-locator-container">
            <div class="map-container">
                <div :id="mapId" class="map"></div>
            </div>

            <div v-if="valueExists" :class="['content', liststyle]">
                <div v-for="key in display" v-if="value[key]" class="content-block">
                    <div class="title">{{ translatedTitle(key) }}</div>
                    <div class="value">{{ value[key] }}</div>
                </div>
            </div>
            <k-empty v-else icon="search" class="k-locator-empty" @click="$refs.input.focus()">
                {{ $t('locator.empty') }}
            </k-empty>
        </div>
    </k-field>
</template>

<script>
import L from "leaflet"
import "leaflet.locatecontrol";

export default {
    data() {
        return {
            map: null,
            marker: null,
            tileLayer: null,
            location: '',
            error: '',
            limit: 1,
            dropdownOptions: [],
            filledStatus: 'closed',
            dragged: false,
            clicked: false,
        }
    },
    props: {
        markerUrl:    String,
        tiles:        String,
        center:       Object,
        zoom:         Object,
        saveZoom:     Boolean,
        autoSaveZoom: Boolean,
        mapbox:       Object,
        display:      Array,
        geocoding:    String,
        liststyle:    String,
        liststyle:    String,
        draggable:    Boolean,
        clickable:    Boolean,
        autocomplete: Boolean,
        language:     [String, Boolean],

        // general options
        label:     String,
        disabled:  Boolean,
        help:      String,
        parent:    String,
        value:     Object,
        name:      [String, Number],
        required:  Boolean,
        type:      String
    },
    computed: {
        mapId() {
            return 'map-'+ this._uid
        },
        icon() {
            return L.divIcon({
                className: 'marker-div-icon',
                iconSize: [20, 20],
                // iconAnchor: [20, 20],
            })
        },
        valueExists() {
            return this.value && (Object.keys(this.value).length > 1 || Object.keys(this.value).length == 1 && !this.value.zoom) ? Object.keys(this.value).length : false
        },
        status() {
            return this.valueExists ? this.filledStatus : ''
        },
        defaultCoords() {
            return this.valueExists ? [this.value.lat, this.value.lon] : [this.center.lat, this.center.lon]
        },
        defaultZoom() {
            return this.valueExists && this.value.zoom ? this.value.zoom : this.zoom.default
        },
        coords() {
            return this.valueExists ? [this.value.lat, this.value.lon] : []
        },
        tileUrl() {
            if(this.tiles == 'mapbox' || this.tiles == 'mapbox.custom') {
                return 'https://api.mapbox.com/styles/v1/'+ this.mapbox.id +'/tiles/256/{z}/{x}/{y}'+ (L.Browser.retina ? '@2x' : '') +'?access_token='+ this.mapbox.token
            }
            else if(this.tiles == 'wikimedia') {
                return 'https://maps.wikimedia.org/osm-intl/{z}/{x}/{y}' + (L.Browser.retina ? '@2x.png' : '.png')
            }
            else if(this.tiles == 'openstreetmap') {
                return 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png'
            }
            else if(this.tiles == 'light_all' || this.tiles == 'voyager') {
                return 'https://cartodb-basemaps-{s}.global.ssl.fastly.net/rastertiles/'+ this.tiles +'/{z}/{x}/{y}' + (L.Browser.retina ? '@2x.png' : '.png')
            }
            else return ''
        },
        attribution() {
            if(this.tiles == 'mapbox' || this.tiles == 'mapbox.custom') {
                return '&copy; <a href="http://www.openstreetmap.org/copyright" target="_blank" rel="noreferrer">OpenStreetMap</a>, &copy; <a href="https://www.mapbox.com/">Mapbox</a>'
            }
            else if(this.tiles == 'wikimedia') {
                return '&copy; <a href="http://www.openstreetmap.org/copyright" target="_blank" rel="noreferrer">OpenStreetMap</a>, &copy; <a href="https://maps.wikimedia.org" target="_blank" rel="noreferrer">Wikimedia</a>'
            }
            else if(this.tiles == 'openstreetmap') {
                return '&copy; <a href="http://www.openstreetmap.org/copyright" target="_blank" rel="noreferrer">OpenStreetMap</a>'
            }
            else if(this.tiles == 'light_all' || this.tiles == 'voyager') {
                return '&copy; <a href="http://www.openstreetmap.org/copyright" target="_blank" rel="noreferrer">OpenStreetMap</a>, &copy; <a href="https://carto.com/attribution" target="_blank" rel="noreferrer">CARTO</a>'
            }
            else return '&copy; <a href="http://www.openstreetmap.org/copyright" target="_blank" rel="noreferrer">OpenStreetMap</a>'
        },
        searchQuery() {
            if(this.geocoding == 'nominatim') {
                let languageParam = this.language ? '&accept-language=' + this.language : ''
                return 'https://nominatim.openstreetmap.org/search?format=jsonv2&limit=1&addressdetails=1&q=' + this.location + languageParam
            }
            else if(this.geocoding == 'mapbox') {
                let languageParam = this.language ? '&language=' + this.language : ''
                return 'https://api.mapbox.com/geocoding/v5/mapbox.places/'+ this.location +'.json?types=address,country,postcode,place,locality&limit='+ this.limit +'&access_token=' + this.mapbox.token + languageParam
            }
            else return ''
        },
    },
    watch: {
        value() {
            this.updateMap()
        }
    },
    mounted() {
        this.initMap()
    },
    methods: {
        onLocationInput() {
            if(!this.autocomplete) return false

            if(this.geocoding && this.location.length) {
                if(this.geocoding != 'mapbox') return false

                this.limit = 5
                fetch(this.searchQuery)
                    .then(response => response.json())
                    .then(response => {
                        // if places are found
                        if(response.features.length) {
                            // keep the relevant ones
                            let suggestions = response.features.filter(el => el.relevance == 1)
                            // make them the dropdown options
                            this.dropdownOptions = suggestions.map(el => {
                                return {
                                    name: el.place_name,
                                    type: this.capitalize(el.place_type[0]),
                                }
                            })
                            this.$refs.dropdown.open()
                        }
                        else {
                            this.$refs.dropdown.close()
                        }
                    })
                    .catch(error => {
                        this.error = this.$t('locator.error')
                        this.$refs.dialog.open()
                        this.$refs.dropdown.close()
                    })
            }
            else {
                this.$refs.dropdown.close()
            }
        },
        select(option) {
            this.location = option.name
            this.getCoordinates()
        },
        translatedTitle(key) {
            key = key.replace('lon', 'longitude')
            key = key.replace('lat', 'latitude')
            return this.$t('locator.'+ key)
        },
        toggle(arg) {
            this.filledStatus = arg
            this.$nextTick(() => {
                this.map.invalidateSize()
                this.map.setView(this.coords, this.defaultZoom)
                if(arg == 'closed' && this.marker) this.disableMapEvents()
                else if(arg == 'open' && this.marker) this.enableMapEvents()
            })
        },
        initMap() {
            // init map
            let zoom = this.value ? this.value.zoom || this.defaultZoom : this.defaultZoom
            this.map = L.map(this.mapId, {minZoom: this.zoom.min, maxZoom: this.zoom.max}).setView(this.defaultCoords, zoom)

            // set the tile layer
            this.tileLayer = L.tileLayer(this.tileUrl, {attribution: this.attribution})
            this.map.addLayer(this.tileLayer)

            // create a marker
            if(this.coords.length) this.setMarker()

            if (this.saveZoom && this.autoSaveZoom) {
                this.map.on('zoomend', () => {
                    this.value = {
                        ...this.value,
                        'zoom': this.map.getZoom()
                    }
                    console.log(this.value)
                    this.$emit("input", this.value)
                    this.dragged = true
                    setTimeout(() => {
                        this.dragged = false
                    }, 500)
                });
            }

            this.locator = L.control.locate({
                position: 'topright',
                strings: {
                    title: "Show me where I am, yo!"
                },
                locateOptions: {
                    enableHighAccuracy: true
                },
                icon: "pin-icon",
                showPopup: false,
                drawMarker: false,
            })
            // this.map.addControl(this.locator)
            //
            this.map.on('locationfound', (e) => {
                
            })

            let setloc = (e) => {
                let _this    = this
                this.coords = e.latlng;
                this.value = {
                    'lat': parseFloat(e.latlng.lat),
                    'lon': parseFloat(e.latlng.lng),
                    'number': null,
                    'city': null,
                    'country': null,
                    'postcode': null,
                    'address': null,
                }

                if(this.saveZoom) {
                    this.value = {
                        ...this.value,
                        'zoom': this.map.getZoom()
                    }
                }

                this.$emit("input", this.value)

                this.clicked = true
                setTimeout(() => {
                    _this.clicked = false
                }, 500)
            }

            if (this.clickable) {
                this.map.on('click', (e) => {
                    setloc(e);
                })
            }

        },
        updateMap() {
            if(this.map) {
                // If a marker already exists
                if(this.marker) {
                    if(this.valueExists) {
                        this.marker.setLatLng(this.coords)
                        if(!this.dragged && !this.clicked) this.toggle('closed')
                    }
                    else {
                        this.map.removeLayer(this.marker)
                        this.marker = null
                    }
                }

                // If a marker should be created
                else if(!this.marker && this.valueExists) {
                    this.setMarker()
                    if(!this.dragged && !this.clicked) this.toggle('closed')
                }

                // If there is a filled value
                if(this.valueExists) {
                    this.map.panTo(this.coords)
                }

                // If there is no filled value, reset default view
                else {
                    this.$nextTick(() => {
                        this.map.invalidateSize()
                        let zoom = this.value ? this.value.zoom || this.defaultZoom : this.defaultZoom
                        this.map.setView(this.defaultCoords, zoom)
                    })
                }
            }
        },
        setMarker() {
            if(this.marker) this.map.removeLayer(this.marker)
            this.marker = L.marker(this.coords, {
                icon: this.icon,
                draggable: this.draggable,
                autoPan: this.draggable,
            })
            this.map.addLayer(this.marker)
            if(this.filledStatus == 'closed') this.disableMapEvents()

            this.marker.on('dragend', e => {
                let position = this.marker.getLatLng()
                let _this    = this

                this.value = {
                    'lat': parseFloat(position.lat),
                    'lon': parseFloat(position.lng),
                    'number': null,
                    'city': null,
                    'country': null,
                    'postcode': null,
                    'address': null,
                }

                if(this.saveZoom) {
                    this.value = {
                        ...this.value,
                        'zoom': this.map.getZoom()
                    }
                }

                this.$emit("input", this.value)
                this.dragged = true
                setTimeout(() => {
                    _this.dragged = false
                }, 500)
            })
        },
        getCoordinates(e) {
            if(e) {
                e.preventDefault()
                e.stopPropagation()
            }

            if(this.$refs.dropdown) this.$refs.dropdown.close()
            this.limit = 1

            if(this.geocoding && this.location.length) {
                fetch(this.searchQuery)
                    .then(response => response.json())
                    .then(response => {
                        if(response.length || Object.keys(response).length) {
                            if(this.geocoding == 'nominatim') {
                                this.setNominatimResponse(response)
                            }
                            else if(this.geocoding == 'mapbox')  {
                                this.setMapboxResponse(response)
                            }
                            this.location = ''
                        }
                        else {
                            this.error = this.$t('locator.empty_response')
                            this.$refs.dialog.open()
                            this.value = {}
                        }

                        this.$emit("input", this.value)
                    })
                    .catch(error => {
                        this.error = this.$t('locator.error')
                        this.$refs.dialog.open()
                    })
            }
        },
        setNominatimResponse(response) {
            response = response[0]
            this.value = {
                'lat': parseFloat(response.lat),
                'lon': parseFloat(response.lon),
                'number': response.address.house_number,
                'city': response.address.city || response.address.town || response.address.village || response.address.county || response.address.state,
                'country': response.address.country,
                'postcode': response.address.postcode,
                'address': response.address.road
            }
            if(this.saveZoom) {
                this.value = {
                    ...this.value,
                    'zoom': this.map.getZoom()
                }
            }
        },
        setMapboxResponse(response) {
            response = response.features[0]
            this.value = {
                'lat':      parseFloat(response.center[1]),
                'lon':      parseFloat(response.center[0]),
                'number':   response.address || '',
                'city':     response.context.find(el => el.id.startsWith('place'))    ? response.context.find(el => el.id.startsWith('place')).text    : '',
                'country':  response.context.find(el => el.id.startsWith('country'))  ? response.context.find(el => el.id.startsWith('country')).text  : '',
                'postcode': response.context.find(el => el.id.startsWith('postcode')) ? response.context.find(el => el.id.startsWith('postcode')).text : '',
                'address':  response.text || ''
            }
            if(this.saveZoom) {
                this.value = {
                    ...this.value,
                    'zoom': this.map.getZoom()
                }
            }
        },
        capitalize(str) {
            return str.charAt(0).toUpperCase() + str.slice(1);
        },
        disableMapEvents() {
            if(this.map) {
                this.map.scrollWheelZoom.disable()
                this.map.dragging.disable()
                this.map.touchZoom.disable()
                this.map.doubleClickZoom.disable()
                this.map.boxZoom.disable()
                this.map.keyboard.disable()
                if (this.map.tap) this.map.tap.disable()
            }
            if(this.marker) this.marker.dragging.disable()
        },
        enableMapEvents() {
            if(this.map) {
                this.map.scrollWheelZoom.enable()
                this.map.dragging.enable()
                this.map.touchZoom.enable()
                if (!this.clickable) this.map.doubleClickZoom.enable()
                this.map.boxZoom.enable()
                this.map.keyboard.enable()
                if (this.map.tap) this.map.tap.enable()
            }
            if(this.marker && this.draggable) this.marker.dragging.enable()
        },
        resetValue() {
            this.value = {}
            this.$emit("input", this.value)
        }
    },
};
</script>

<style lang="scss">
    @import '../assets/css/styles.scss'
</style>
