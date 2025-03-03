<template>
    <div style="min-height: 450px">
        <ReportsSubmenu></ReportsSubmenu>

        <div class="wrapper__submenu--content">
        <h2>{{ $t('global.Classes') }}</h2>

        <ReportPeriodWithTeacherClasstypeList v-bind.sync="selectedPeriod"></ReportPeriodWithTeacherClasstypeList>

        <loading-animation v-if="loading"></loading-animation>

        <div v-else>
            <div class="toolbar">
                <md-button md-theme-default class="md-primary md-raised ml-0" @click="downloadFile('csv')" :disabled="selectedPeriod.invalidDuration">Download CSV</md-button>
                <md-button md-theme-default class="md-primary md-raised ml-0" @click="downloadFile('xlsx')" :disabled="selectedPeriod.invalidDuration">Download XLSX</md-button>
                <md-button md-theme-default class="md-primary md-raised ml-0" @click="downloadFile('pdf')" :disabled="selectedPeriod.invalidDuration">Download PDF</md-button>          
            </div>

            <div v-if="!selectedPeriod.invalidDuration" class="w-100 overflow-scroll">
                <table class="classes">
                    <tr>
                        <th>{{ $t('global.ID') }}</th>
                        <th>{{ $t('global.Date') }}</th>
                        <th>{{ $t('global.Start') }}</th>
                        <th>{{ $t('global.End') }}</th>
                        <th>{{ $t('global.Duration') }}</th>
                        <th>{{ $t('global.Class') }}</th>

                        <th>{{ $t('global.Teacher') }}</th>

                        <th>{{ $t('global.Room') }}</th>

                        <th v-if="branches">{{ $t('global.Branch') }}</th>
                        <th v-if="livestream_enabled">{{ $t('global.PhysicalAttendance') }}</th>
                        <th v-if="livestream_enabled">{{ $t('global.Livestream') }}</th>
                        <th v-if="classpass_com_integration_enabled">{{ $t('global.ClassPassComEnabled') }}</th>
                        <th>{{ $t('global.Cancelled') }}</th>
                        
                        <th>{{ $t('global.SignUps') }}</th>
                        <th>{{ $t('global.CheckedIn') }}</th>
                        <th v-if="livestream_enabled">{{ $t('global.LivestreamSignups') }}</th>
                        
                        <th v-if="classpass_com_integration_enabled">ClassPass.com {{ $t('global.SignUps') }}</th>

                    </tr>
                    <tr v-for="classItem in classes">
                        <td>{{ classItem.id }}</td>
                        <td>{{ dbDateToHumanDate(classItem.date) }}</td>
                        <td>{{ classItem.start_time }}</td>
                        <td>{{ classItem.end_time }}</td>
                        <td>{{ getDuration(classItem.start_time, classItem.end_time) }}</td>
                        <td>{{ classItem.class_type.name }}</td>
                        <td>{{ classItem.teachersList }}</td>
                        <td>{{ classItem.room.name }}</td>
                        <td v-if="branches">{{ classItem.room.branch.name }}</td>
                        
                        <td v-if="livestream_enabled">
                            <span v-if="classItem.studio_attendance_enabled == 1">{{ $t('global.Yes') }}</span>
                            <span v-else>{{ $t('global.No') }}</span>
                        </td>
                        
                        <td v-if="livestream_enabled">
                            <span v-if="classItem.livestream_enabled == 1">{{ $t('global.Yes') }}</span>
                            <span v-else>{{ $t('global.No') }}</span>
                        </td>
                        
                        <td v-if="classpass_com_integration_enabled">
                            <span v-if="classItem.classpass_com_enabled == 1">{{ $t('global.Yes') }}</span>
                            <span v-else>{{ $t('global.No') }}</span>
                        </td>
                        
                        <td v-if="classItem.cancelled == 1">{{ $t('global.Yes') }}</td>
                        <td v-else>{{ $t('global.No') }}</td>

                        <td>{{ classItem.signup_count }}</td>
                        <td>{{ classItem.checkedin_count }}</td>
                        <td v-if="livestream_enabled">{{ classItem.livestream_signup_count }}</td>
                        <td v-if="classpass_com_integration_enabled">{{ classItem.classpass_signup_count }}</td>
                        
                    </tr>
                    <tr v-if="!classes.length">
                        <td colspan="5">{{ $t('global.NoClassesForTheSelectedTimePeriod') }}</td>
                    </tr>

                    <tr>
                        <th colspan="4">
                            <md-button v-if="classes.length" @click="toggleFolded(idx)" v-bind:style="{margin: '0px', fontWeight: 'bold', color: 'white'}">
                            {{ $t('global.Total') }}: {{classes.length}} classes
                            <md-icon v-if="folded" v-bind:style="{color: 'white'}">expand_more</md-icon>
                            <md-icon v-else v-bind:style="{color: 'white'}">expand_less</md-icon>
                            </md-button>
                            <span v-else>{{ $t('global.Total') }}:</span>
                        </th>   

                        <th v-if="classes.length">
                            {{ durationStringWithoutSecond(totalMins) }}
                        </th>                        
                        <th v-else></th>

                        <th></th>
                        <th></th>
                        <th></th>
                        <th v-if="branches"></th>
                        <th v-if="livestream_enabled"></th>
                        <th v-if="livestream_enabled"></th>
                        <th v-if="classpass_com_integration_enabled"></th>
                        <th></th>
                        
                        <th v-if="classes.length">{{ totalSignups }}</th>
                        <th v-else></th>

                        <th v-if="classes.length">{{ totalCheckedIn }}</th>
                        <th v-else></th>
                        
                        <th v-if="livestream_enabled">
                            <span v-if="classes.length">{{ totalLivestreamSignups }}</span>
                            <span v-else></span>
                        </th>
                        
                        <th v-if="classpass_com_integration_enabled"></th>
                    </tr>
                </table>
            </div>
        </div>

    </div>

    <div class="space8"></div>
    <div class="space8"></div>
    <div class="space8"></div>
    <div class="space8"></div>
    <div class="space8"></div>

    <md-dialog :md-active.sync="showParticipantsDialog" v-if="classParticipantsDialogClass">
        <md-dialog-title>{{ classParticipantsDialogClass.class_type.name }},
            {{ dbDateToHumanDate(classParticipantsDialogClass.date) }},
            {{ classParticipantsDialogClass.start_time.substr(0, 5) }} -
            {{ classParticipantsDialogClass.end_time.substr(0, 5) }}
        </md-dialog-title>
        <md-dialog-content>
            <table class="participants">
                <tr>
                    <th>{{ $t('global.Name') }}</th>
                    <th>{{ $t('global.StartEndDuration') }}</th>
                    <th>System</th>
                </tr>
                <tr v-for="(user, idx) in classLivestreamUsers" :class="{teacher: user.sessions[0].role === 'teacher'}">
                    <td>{{ user.first_name }} {{ user.last_name }}</td>
                    <td :class="{'no-padding': true}">
                        <table class="sessions">
                            <tr v-for="session in user.sessions">
                                <td>{{ session.start }}</td>
                                <td>{{ session.end }}</td>
                                <td>{{ durationString(session.livestream_seconds) }}</td>
                            </tr>
                        </table>
                    </td>
                    <td>
                        <span v-if="user.parsedUserAgent">
                            <span v-if="user.parsedUserAgent.device.model">
                            {{ user.parsedUserAgent.device.vendor }}
                            {{ user.parsedUserAgent.device.model }}
                            <br>
                            </span>
                            <span v-if="user.parsedUserAgent.browser.name === 'WebKit'">
                            YOGO App
                            </span>
                            <span v-else>
                            {{ user.parsedUserAgent.browser.name }}
                            {{ user.parsedUserAgent.browser.major }}
                            </span>
                            <br>
                            {{ user.parsedUserAgent.os.name }}
                            {{ user.parsedUserAgent.os.version }}

                        </span>
                    </td>
                </tr>
            </table>
        </md-dialog-content>
        <md-dialog-actions>
            <md-button type="button" @click.prevent="showParticipantsDialog = false">
                {{ $t('global.Close') }}
            </md-button>
        </md-dialog-actions>
        </md-dialog>
    </div>
</template>

<script>

    import ReportPeriodWithTeacherClasstypeList from './ReportPeriodWithTeacherClasstypeList';

    import LoadingAnimation from './LoadingAnimation';
    import YogoApi from '../gateways/YogoApi';
    import { mapGetters } from 'vuex';

    import moment from 'moment-timezone';

    import ReportsSubmenu from './ReportsSubMenu';

    import dateTimeFunctions from '../mixins/dateTimeFunctions';
    import downloadFile from '../helpers/downloadFile';

    const REPORT_AVAILABLILITY_DATE = '2021-02-12';

    export default {
        name: 'ReportClasses',
        mixins: [dateTimeFunctions],
        components: {
            ReportsSubmenu,
            LoadingAnimation,
            ReportPeriodWithTeacherClasstypeList,
        },
        data() {
            return {
                selectedPeriod: {
                    fromDate: moment.tz('Europe/Copenhagen')
                        .toDate(),
                    endDate: moment.tz('Europe/Copenhagen')
                        .toDate(),
                    dateUpdated: false,
                    dataUpdated: false,
                    invalidDuration: false,
                    teachers: [],
                    classTypes: [],
                    onlyPhysicalAttendance: false,
                    onlyLivestream: false,
                    onlyClassPassEnabled: false,
                    countClassTypes: 0,
                },

                teachers: [],

                classes: [],
                loading: true,               

                classParticipantsDialogClass: null,
                classLivestreamUsers: [],
                showParticipantsDialog: false,

                classpass_com_integration_enabled: false,
                livestream_enabled: false,
                branches: false,

                totalMins: 0, 
                totalCheckedIn: 0, 
                totalSignups: 0, 
                totalLivestreamSignups: 0,
                folded: false,
            };
        },
        computed: {
            ...mapGetters([
                'client',
                'stateReady',
                'apiRoot',
            ]),
            totalLivestreamSeconds() {
                return _.sum(
                    _.map(this.classes, 'totalSeconds'),
                );
            },
            totalParticipantSessions() {
                return _.sum(
                    _.map(this.classes, 'numberOfParticipants'),
                );
            },
        },
        watch: {
            stateReady(newReadyState) {
                if (newReadyState) this.fetchData();
            },
            selectedPeriod: {
                handler: function (newPeriod, oldPeriod) {
                    if ( newPeriod.dataUpdated ) {
                        if (!newPeriod.invalidDuration) {
                            this.fetchData(newPeriod.dateUpdated);
                        }
                    }                    
                },
                deep: true,
            },
        },
        mounted() {     
            this.fetchClasspassSettings();
            this.fetchBranch();       
            if (this.stateReady) this.fetchData();
        },

        methods: {
            async fetchClasspassSettings() {
                this.loading = true;
                const res = await YogoApi.get('/clients/' + this.client.id + '/settings/?keys[]=livestream_enabled&keys[]=classpass_com_integration_enabled');

                this.loading = false;
                this.classpass_com_integration_enabled = res.classpass_com_integration_enabled;
                this.livestream_enabled = res.livestream_enabled;
            },
            async fetchBranch() {
                const branches = await YogoApi.get('/branches/');
                this.branches = branches.length > 1 ? true: false;
            },
            async fetchData(dateUpdated) {
                this.loading = true
                if (this.selectedPeriod.fromDate <= this.selectedPeriod.endDate) {
                    let allClasses = await YogoApi.get('/classes' +
                        '?startDate=' + moment(this.selectedPeriod.fromDate).format('YYYY-MM-DD') +
                        '&ignoreArchived=1' +
                        '&endDate=' + moment(this.selectedPeriod.endDate).format('YYYY-MM-DD') +
                        '&populate[]=class_type' +
                        '&populate[]=teachers' +
                        '&populate[]=room' +
                        '&populate[]=room.branch' +
                        '&populate[]=signup_count' +
                        '&populate[]=waiting_list_count' +
                        '&populate[]=waiting_list_max' +
                        '&populate[]=livestream_signup_count' +
                        '&sort[]=' + encodeURIComponent('date ASC') +
                        '&sort[]=' + encodeURIComponent('start_time ASC') ,
                    )
                    this.classes = allClasses.classes
                    this.classes = _.sortBy(this.classes, ['date', 'start_time'])

                    if (this.selectedPeriod.teachers.selectedAll) {
                        this.teachers = await YogoApi.get(
                            '/users' +
                            '?teacher=1' +
                            '&ignoreArchived=1' +
                            '&populate[]=image',
                        );
                    } else {
                        if (this.selectedPeriod.teachers.teachers) {
                            this.teachers = this.selectedPeriod.teachers.teachers.map(teacher => teacher);
                        } else {
                            this.teachers = [];
                        }
                    }
                } else {
                    this.classes = [];
                    this.teachers = [];
                }

                this.totalMins = 0;
                this.totalCheckedIn = 0;
                this.totalSignups = 0;
                this.totalLivestreamSignups = 0;
                console.log("teachers = ", this.teachers);
                
                for (const j in this.classes) {

                    let exist = false;
                    if (this.selectedPeriod.onlyPhysicalAttendance && !this.classes[j].studio_attendance_enabled) continue;
                    if (this.selectedPeriod.onlyLivestream && !this.classes[j].livestream_enabled) continue;
                    if (this.selectedPeriod.onlyClassPassEnabled && !this.classes[j].classpass_com_enabled) continue;

                    if (!this.selectedPeriod.classTypes.selectedAll) {
                        for (const k in this.selectedPeriod.classTypes.classTypes) {
                            if (this.classes[j].class_type_id == this.selectedPeriod.classTypes.classTypes[k].id) {
                                exist = true;
                                break;
                            }
                        }

                        if (!exist) continue;
                    }
                    
                    for (const k in this.classes[j].teachers) {
                        for (const i in this.teachers) {
                            if (this.teachers[i].id == this.classes[j].teachers[k].id) {
                                //get teachers list
                                let teachersList = "";
                                for (const kk in this.classes[j].teachers) {
                                    if (teachersList != "") {teachersList += ", ";}
                                    teachersList += this.classes[j].teachers[kk].first_name + " " + this.classes[j].teachers[kk].last_name;
                                }
                                this.classes[j].teachersList = teachersList;
                                break;
                            }
                        }
                    }

                    let start_timer = parseInt(this.classes[j].start_time.split(":")[0]) * 60 + parseInt(this.classes[j].start_time.split(":")[1])
                    let end_timer = parseInt(this.classes[j].end_time.split(":")[0]) * 60 + parseInt(this.classes[j].end_time.split(":")[1])
                    this.totalMins += end_timer - start_timer
                    this.totalCheckedIn += this.classes[j].checkedin_count;
                    this.totalSignups += this.classes[j].signup_count;
                    this.totalLivestreamSignups += this.classes[j].livestream_signup_count;
                }
                this.selectedPeriod.dataUpdated = false;
                this.loading = false
            },

            durationString(seconds) {
                return [_.padStart(Math.floor(seconds / 3600), 2, '0') +
                    this.$t('time.hours') ,
                    _.padStart(Math.floor((seconds % 3600) / 60), 2, '0') +
                    this.$t('time.minutes') ,
                    _.padStart(seconds % 60, 2, '0') +
                    this.$t('time.seconds')].join(" ");
            },
            durationStringWithoutSecond(mins) {
                const hh = Math.floor(mins / 60);
                const mm = Math.floor(mins % 60);
                return _.padStart(hh, 2, '0') + ":" + _.padStart(mm, 2, '0')
            },

            getDuration(start_time, end_time) {
                let start_timer = parseInt(start_time.split(":")[0]) * 60 + parseInt(start_time.split(":")[1]);
                let end_timer = parseInt(end_time.split(":")[0]) * 60 + parseInt(end_time.split(":")[1]);
                let mins = end_timer - start_timer;
                const hh = Math.floor(mins / 60);
                const mm = Math.floor(mins % 60);
                return _.padStart(hh, 2, '0') + ":" + _.padStart(mm, 2, '0')
            },

            async showClassUsers(classItem) {
                this.classParticipantsDialogClass = classItem;
                const classLivestreamSessions = await YogoApi.get('/reports/livestream/class/' + classItem.id);
                this.classLivestreamUsers = _.chain(classLivestreamSessions)
                    .groupBy('user.id')
                    .map((userSessions) => {
                        const user = userSessions[0].user;
                        user.sessions = userSessions;
                        user.sortByTeacher = user.sessions[0].role !== 'teacher';
                        user.parsedUserAgent = user.sessions[0].parsedUserAgent;
                        return user;
                    })
                    .toArray()
                    .sortBy('sortByTeacher', 'first_name', 'last_name')
                    .value();
                this.showParticipantsDialog = true;
            },

            toggleFolded() {
                this.folded = !this.folded;
            },

            async downloadFile(format) {
                console.log("data = ", {
                        teachers: this.teachers.map(teacher => {return {id: teacher.id, name: teacher.first_name + " " + teacher.last_name}}),
                        classTypes: this.selectedPeriod.classTypes.classTypes.map(classType => {return {id: classType.id, name: classType.name}}),
                        fromDate: this.selectedPeriod.fromDate,
                        endDate: this.selectedPeriod.endDate,
                        allClassTypes: this.selectedPeriod.classTypes.selectedAll,
                        onlyPhysicalAttendance: this.selectedPeriod.onlyPhysicalAttendance,
                        onlyLivestream: this.selectedPeriod.onlyLivestream,
                        onlyClassPassEnabled: this.selectedPeriod.onlyClassPassEnabled,
                    })
                const response = await YogoApi.post(
                    '/reports/make-report-token',
                    {
                        teachers: this.teachers.map(teacher => {return {id: teacher.id, name: teacher.first_name + " " + teacher.last_name}}),
                        classTypes: this.selectedPeriod.classTypes.classTypes.map(classType => {return {id: classType.id, name: classType.name}}),
                        fromDate: this.selectedPeriod.fromDate,
                        endDate: this.selectedPeriod.endDate,
                        allClassTypes: this.selectedPeriod.classTypes.selectedAll,
                        onlyPhysicalAttendance: this.selectedPeriod.onlyPhysicalAttendance,
                        onlyLivestream: this.selectedPeriod.onlyLivestream,
                        onlyClassPassEnabled: this.selectedPeriod.onlyClassPassEnabled,
                    },
                );

                const relativePath = '/reports/class?format=' + format + '&reportToken=' + response.token;

                downloadFile(relativePath);

            },

            moment: moment,
        },
    };
</script>


<style lang="scss" scoped>

    @import '../assets/scss/variables.scss';

    table.classes, table.participants {
        border-collapse: collapse;
        width: 100%;
        background: #fff;
        border: 1px solid #ddd;;

        th, td {
            text-align: left;
            line-height: 1.4;
            font-weight: normal;
            border: 1px solid #ddd;
            padding: 12px 15px;
        }

        td.no-padding {
            padding: 0;
        }

        th {
            font-weight: bold;
            background: #01a7c2;
            color: #fff;
        }

        &.participants tr.teacher {
            td {
                font-weight: bold;
                background: #f5f5f5;

                a {
                    font-weight: bold;
                }
            }
        }

    }

    table.sessions {
        border-collapse: collapse;
        width: 100%;
        background: #fff;

        th, td {
            text-align: left;
            line-height: 1.2;
            font-weight: normal;
            border: none;
            padding: 6px 8px;
            color: rgba(0, 0, 0, 0.87);
        }

        th {
            background: #ccc;
        }
    }

    table.classes th {
        text-align: center !important;
    }


</style>
