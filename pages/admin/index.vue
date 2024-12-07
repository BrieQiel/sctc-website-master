<template>
<div>
  <v-row no-gutters justify="center" v-if="loginFormDialog === false">
    <v-col cols="12">
      <!-- Start: Table for Registered Participants -->
      <v-card class="mx-auto" variant="outlined" width="1300">
        <v-data-table :headers="item_headers" :items="listOfParticipants" >
          <template v-slot:top>
            <v-toolbar density="compact" color="#ff7b02">
              <v-toolbar-title> List of Registered Participants </v-toolbar-title>
              <v-spacer></v-spacer>
              <v-btn variant="elevated" color="green" @click="exportToExcel" class="btn btn-default">
                Download To Excel
              </v-btn>
            </v-toolbar>
          </template>


          <template v-slot:[`item.event_type`]="{value}">
            <v-chip color="#ff7b02">
              {{ value }}
            </v-chip>
          </template>

          <template v-slot:[`item.status`]="{value}">
            <v-chip color="orange" v-if="value">{{value}}</v-chip>
          </template>

          <template v-slot:[`item.action`]="{item}">
            <v-btn size="small" @click="send_confirmation(item)">Send</v-btn>
          </template>

        </v-data-table>
      </v-card>
      <!-- Start: Table for Registered Participants -->
    </v-col>

    <v-col cols="12">
      <!-- Start: Table for Invited Participants -->
      <v-card class="mx-auto mt-2" variant="outlined" width="1300">
        <v-data-table :headers="invited_headers" :items="listOfInvitedParticipants">
          <template v-slot:top>
            <v-toolbar density="compact" color="#ff7b02">
              <v-toolbar-title> List of Invited Participants </v-toolbar-title>
              <v-spacer></v-spacer>
              <v-btn variant="elevated" @click="inviteDialog = true"> Invite Participants </v-btn>
            </v-toolbar>
          </template>

          <template v-slot:[`item.event_type`]="{value}">
            <v-chip color="#ff7b02">
              {{ value }}
            </v-chip>
          </template>

          <template v-slot:[`item.status`]="{value}">
            <v-chip :color="value ? '#ff7b02' : 'green'">
              {{ value ? value : 'Pending'}}
            </v-chip>
          </template>

          <template v-slot:[`item.action`]="{item}">
            <v-btn size="small" @click="send_reminder(item)">Send</v-btn>
          </template>

          
        </v-data-table>   
      </v-card>
      <!-- End: Table for Invited Participants -->
    </v-col>
  </v-row>

  <v-dialog v-model="inviteDialog" width="500">
    <v-card>
      <v-toolbar density="compact" color="#ff7b02">
        <v-toolbar-title> Fill in the details of the participants </v-toolbar-title>
        <v-spacer></v-spacer>
        <v-btn icon @click="inviteDialog = false">
          <v-icon icon="mdi-close" />
        </v-btn>
      </v-toolbar>
      <v-card-text>
        <v-form ref="invitationForm" v-model="isFormValid">
          <v-text-field
            required
            @click.right.prevent
            @copy.prevent
            @paste.prevent
            variant="outlined"
            label="Firstname"
            v-model="form.firstname"
            :rules="[v => !!v || 'Firstname is required']"
          />

          <v-text-field
            required
            @click.right.prevent
            @copy.prevent
            @paste.prevent
            variant="outlined"
            label="Surname"
            v-model="form.lastname"
            :rules="[v => !!v || 'Surname is required']"
          />

          <v-text-field
            required
            @click.right.prevent
            @copy.prevent
            @paste.prevent
            variant="outlined"
            label="Company Name"
            v-model="form.company"
            :rules="[v => !!v || 'Company Name is required']"
          />

          <v-text-field
            required
            @click.right.prevent
            @copy.prevent
            @paste.prevent
            variant="outlined"
            label="Job Position"
            v-model="form.position"
            :rules="[v => !!v || 'Position is required']"
          />

          <v-text-field
            required
            @click.right.prevent
            @copy.prevent
            @paste.prevent
            variant="outlined"
            label="Email Address"
            v-model="form.email"
            :rules="[v => !!v || 'Email is required', v => /.+@.+\..+/.test(v) || 'Email must be valid']"
          />
        </v-form>
      </v-card-text>
      <v-card-actions>
        <v-btn
          variant="elevated"
          class="text-white"
          color="orange"
          :disabled="!isFormValid"
          @click="inviteParticipants"
        >
          <v-icon left icon="mdi-email-fast-outline" class="mr-2" />
          Send Invitation
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>

  <v-dialog persistent v-model="loginFormDialog" width="400">
    <v-card >
      <v-toolbar color="#ff7b02">
        <v-toolbar-title> Login </v-toolbar-title>
        <v-spacer></v-spacer>
      </v-toolbar>
      <v-card-text>
        <v-text-field  variant="outlined" label="Username" v-model="loginform.username"/>
        <v-text-field type="password" variant="outlined" label="Password" v-model="loginform.password" @keyup.enter="loginAdmin"/>
      </v-card-text>
      <v-card-actions>
        <v-btn @click="loginAdmin">Login</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</div>

</template>

<script setup lang="ts">
import ExcelJS from "exceljs";
import { useRoute } from 'vue-router';
import { getParticipants, sendMail, login } from '../../../service/mailer.ts';
import {eventsData} from '../../data/content/events.content.ts'
import { MAILER_ENDPOINT } from '../../config/config.ts';
import Swal from 'sweetalert2';
import axios from 'axios';
const {$rest} = useNuxtApp()
useHead({ title: 'Admin' });
const route = useRoute();


onBeforeMount(async () => {
  
  Promise.all([
    loadEventData(),
    loadParticipants(),
    loadInvitedParticipants()
  ]).then(() => {
    loader.value = false
  })
});

const loader = ref(true)



//Start: Load the existing data
const eventData = ref({});
const loadEventData = async () => {
  return eventData.value = eventsData.find((event) => event.current);
}
//End: Load the existing data

// Start: Load the participants
const listOfParticipants = ref([]);
const loadParticipants = async () => {
  const { data } = await getParticipants();
  return listOfParticipants.value = data;
}
// End: Load the participants

// Start: Get invited participants
const listOfInvitedParticipants = ref([])
async function loadInvitedParticipants() {
  return axios.get(`${MAILER_ENDPOINT}/mailer/getInvitedParticipants`)
      .then((response: any) => {
        console.log('asdsa', response.data)
        return  listOfInvitedParticipants.value = response.data
      })
}

//Start: Invitation Form Dialog
const inviteDialog = ref(false);
const isFormValid = ref(false);
const form = ref({
  event_type: '',
  firstname : '',
  lastname  : '',
  company   : '',
  position  : '',
  email     : '',
  number   : ''
});
// End: Invitation Form Dialog

// Start: Add Invited Participants to DB
async function addInvitedParticipants(){
  try {
    const response = await axios.post(`${MAILER_ENDPOINT}/mailer/saveInvitedParticipants`, {
      ...form.value,
      event_type: eventData.value.eventType,
    });
    
    console.log('Response received:', response);
    return response; 
  } catch (error) {

    console.log(Error)
  }
}
// End: Add Invited Participants to DB

function inviteParticipants(){
  
  const emailPayload = {
    eventName: eventData.value.register.eventName,
    eventDeadline: eventData.value.register.deadline,
    receivingEmail: eventData.value.register.registerEmail || null,
    fromName: eventData.value.register.fromName,
    emailContent: eventData.value.invitationContent,
    ...form.value,
  };
  
   addInvitedParticipants()
    .then((response) => {
      loadInvitedParticipants()
      sendMail(emailPayload, 'hcd_invitation', false);
      Swal.fire({
      title: "Sent Email",
      icon: "success",
      showConfirmButton: false,
      willOpen: () => {
        // Dynamically adjust z-index if needed
        const swalElement = document.querySelector('.swal2-container');
        if (swalElement) {
          swalElement.style.zIndex = '99999'; // Set it to a very high value
        }
      }
    });
    })
    .catch((error) => {
      console.error('Error adding participants:', error);
    });
}

function send_reminder(items: any){
  return sendMail(items, 'hcd_reinvite', false).then(() => {
    console.log('Succ')
  }).catch((error: any) => {
    console.log(error)
  })
}

function send_confirmation(items: any){
  return sendMail(items, 'hcd_reminder', false).then(() => {
    console.log('Succ')
  }).catch((error: any) => {
    console.log(error)
  })
}

// Table headers
const item_headers = ref([
  { title: 'Type of Event', key: 'event_type' },
  { title: 'Firstname',     key: 'firstname' },
  { title: 'Lastname',      key: 'lastname' },
  { title: 'Company',       key: 'company' },
  { title: 'Position',      key: 'position' },
  { title: 'Email',         key: 'email' },
  { title: 'Number',         key: 'number' },
  { title: 'Status',         key: 'status' },
  { title: 'Action',        key: 'action' }, 
].map(v => ({ ...v, align: 'center', sortable: false})));

const invited_headers = ref([
  { title: 'Type of Event', key: 'event_type' },
  { title: 'Firstname',     key: 'firstname' },
  { title: 'Lastname',      key: 'lastname' },
  { title: 'Company',       key: 'company' },
  { title: 'Position',      key: 'position' },
  { title: 'Email',         key: 'email' },
  // { title: 'Status',         key: 'status' },
  { title: 'Action',        key: 'action' }, 
].map(v => ({ ...v, align: 'center', sortable: false})));

// Start: Login Form
const loginFormDialog = ref(true)
const loginform = ref({
  username: '',
  password: ''
})

async function loginAdmin(){
  await login({
    ...loginform.value
  }).then(() => {
    loader.value = false
    loginFormDialog.value = false
    Swal.fire({
      title: "Success",
      icon: "success",
      willOpen: () => {
        // Dynamically adjust z-index if needed
        const swalElement = document.querySelector('.swal2-container');
        if (swalElement) {
          swalElement.style.zIndex = '99999'; // Set it to a very high value
        }

      }
    });

  }).catch(() => {
    Swal.fire({
      title: "Invalid Username or Password",
      icon: "error",
      willOpen: () => {
        // Dynamically adjust z-index if needed
        const swalElement = document.querySelector('.swal2-container');
        if (swalElement) {
          swalElement.style.zIndex = '99999'; // Set it to a very high value
        }
      }
    });
  })
}

function exportToExcel() {
  const workbook = new ExcelJS.Workbook();
  const worksheet = workbook.addWorksheet("Sheet 1");

  worksheet.columns = [
    {
      header: "First Name",
      key: "firstname",
      width: 30,
    },
    {
      header: "Last Name",
      key: "lastname",
      width: 30,
    },
    {
      header: "Company",
      key: "company",
      width: 50,
    },
    {
      header: "Position",
      key: "position",
      width: 70,
    },
    {
      header: "Number",
      key: "number",
      width: 30,
    },
    { 
      header: "Email", 
      key: "email", width: 50 
    },
    { 
      header: "Status", 
      key: "status", 
      width: 30 },
  ];

  // Apply styles to the header row only
  worksheet.getRow(1).font = { name: "Arial Black", bold: true };
   

  // Assuming `listOfInvitedParticipants.value` is an array of participant objects
  listOfParticipants.value.forEach((obj: any) => {
    worksheet.addRow({
      firstname: obj.firstname,
      lastname: obj.lastname,
      company: obj.company,
      position: obj.position,
      number: obj.number,
      email: obj.email,
      status: obj.status,
    });
  });

  workbook.xlsx.writeBuffer().then((buffer) => {
    const blob = new Blob([buffer], {
      type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    });
    const link = document.createElement("a");
    link.href = window.URL.createObjectURL(blob);
    link.download = `participants.xlsx`;
    link.click();
  });
}

</script>
<style scoped>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%; /* Full height to ensure centering */
}

.swal2-custom-zindex {
  z-index: 99999 !important; /* Ensure this value is higher than v-dialog's z-index */
}
</style>
