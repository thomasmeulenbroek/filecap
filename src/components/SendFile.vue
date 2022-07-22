<template>
<h1>Send File </h1>
    <form id="sendFileForm" action="#">
        <div id="files">
            <label for="files">Selecteer bestand(en) om te uploaden</label>
            <input type="file" name="files" multiple v-on:change="updateFiles($event)"/>
        </div>
        <div id="emails">
            <div>
                <label for="subject">Onderwerp</label>
                <input type="text" name="subject" v-model="subject"/>
            </div>

            <div>
                <label for="comment">Comment</label>
                <textarea name="comment" v-model="comment" multiline></textarea>
            </div>

            <div>
                <label for="email_to">Ontvanger</label>
                <input type="text" name="email_to" v-model="recipient"/>
            </div>

            <div>
                <label for="email_from">Afzender</label>
                <input type="text" name="email_from" v-model="sender"/>
            </div>
        </div>
        <div class="password_input">
            <div>
                <label for="password">Vul wachtwoord in</label>
                <input type="password" name="password" v-model="password"/>
            </div>

            <div>
                <label for="password_repeat">Herhaal wachtwoord</label>
                <input type="password" name="password_repeat" v-model="repeat"/>
            </div>
            </div>
        <div class="error" v-for="error in errors" :key="error">
             <li>{{ error }}</li>
        </div>
        <input type="button" value="Versturen" v-on:click="sendFiles()" />
    </form>
</template>

<style scoped>
.error{
    color:red
}
div {
    margin: 5px
}
</style>

<script>
const hostname = import.meta.env.VITE_FILECAP_HOSTNAME
const apiKey = import.meta.env.VITE_API_KEY
const sendaddress = import.meta.env.VITE_SEND_ADDRESS
export default {
    data(){
        return {
            files: [],
            requirements : null,
            subject: 'Test Subject',
            comment: 'Test Comment',
            password : 'aaaaaD55555',
            repeat: 'aaaaaD55555',
            sender: sendaddress,
            recipient: 'thmeulenbroek@gmail.com',
            errors: [],
            storeDays: 5,
            downloadLoadable_value: 10
        }
    },
    methods: {
        /**
         * Get password requirements from the filecap server
         */
        getPasswordRules(){        
            const options = {
                method: 'POST',
                headers: {
                    'Content-Type' : 'application/json',
                },
                body: JSON.stringify( {
                    apiKey: apiKey,
                    lang : 'NL'
                })
            }    

            fetch(`${hostname}/FileCap/api/settings/getPasswordRules`, options)
            .then( response => response.json())
            .then( data => this.requirements = data.value.rules)
            .catch(error => this.errors.push(`Error fetching password rules: ${error.message}`))
        },
        /**
         * Validate the password against the password rules and check if both passwords match
         */
        validatePasswords(){
            let errors = []
            console.log(this.requirements)
            if(this.requirements == null){
                return this.errors.push('Geen wachtwoord regels gevonden... probeer later opnieuw')
            }
            for(let i = 0; i < this.requirements.length; i++){
               if(! this.password.match(this.requirements[i].regex)){
                errors.push(this.requirements[i].errorMessage)
               }
            }
            if(this.password.localeCompare(this.repeat) !== 0){
                errors.push('Wachtwoorden komen niet overeen')
               }
           this.errors =[...this.errors, ...errors]
        },
        /**
         * Validate if emails are valid and make sure the sender is from @centcom.network
         */
        validateEmails(){
           if(!/(.+)@(.+){2,}\.(.+){2,}/.test(this.recipient)){
                this.errors.push('Ongeldig ontvanger adres')
           }
           if(!/(.+)@centcom.network/.test(this.sender)){
                this.errors.push('Afzender moet van @centcom.network zijn')
           }
        },
        /**
         * 
         */
        updateFiles(event){
            this.files = event.target.files
        },
        /**
         * Verify to make sure that each selected file is allowed to be uploaded to the filcap server
         */
        async checkFiles(){
            for(let i = 0; this.files.length > i; i++){

                //create formdata
                const formData = new URLSearchParams()
                formData.append('APIKey', apiKey)
                formData.append('filename', this.files[i].name)
                formData.append('mime-type', this.files[i].type)
                formData.append('sender', this.sender)

                let options = {
                    method: 'POST',
                    headers: {
                        'Content-Type' : 'application/x-www-form-urlencoded'
                    },
                    body: formData
                }   

                try {
                    //valid will be true if file is allowed to be uploaded to the server
                    const valid = await fetch(`${hostname}/FileCap/checkFile.jsp`, options)
                        .then(response => response.text())
                        .then(data => {
                        const parser = new DOMParser()
                        const doc = parser.parseFromString(data, 'application/xml')
                        return doc.querySelector('valid').innerHTML })
                        .catch(error => this.errors.push(`Error checking for valid file upload: ${error}`))
                    
                    //add error message if file is not allowed to be uploaded
                    if(valid === "False"){
                        this.errors.push(`Bestand: ${this.files[i].name} is een ongeldig bestandformaat`)
                    }
                
                } catch (error){
                    this.errors.push(this.errors.push(`Error checking for valid file upload: ${error.message}`))
                }
            }
        },
        /**
         * Main method that checks user input and sends the files to the FileCapAPI
         */
        sendFiles(){
            this.errors = []
            //validate user input
            this.validateEmails()
            this.validatePasswords()
            
            if(this.files.length == 0)
               return this.errors.push("Selecteer eerst bestanden om te uploaden")
            
            this.checkFiles()
            if(this.errors.length > 0)
                return 
            
            //no errors left, files can be uploaded

            const formData = new FormData()
            formData.append('APIKey', apiKey)
            formData.append('from',this.sender)
            formData.append('subject',this.subject)
            formData.append('comment', this.comment)
            formData.append('password', this.password)
            formData.append('rec0', this.recipient )
            formData.append('storeDays_value', this.storeDays)
            formData.append('downloadLoadable_value', this.downloadLoadable_value)
            
            //add all files to the formdata
            for(let i = 0; this.files.length > i; i++){
                formData.append(`file${i}`,this.files[i])
            }
            
            const options = {
                method: 'POST',
                body: formData
            }

            fetch(`${hostname}/FileCap/process_upload.jsp`, options)
                .then(response => response.json())
                .then(data => {
                    //request returns true if all files are uploaded correctly
                    if(!data){
                        this.errors.push(data)
                    }
                })
                .catch(error => this.errors.push(`Error uploading files: ${error}`))
        }
    },
    /**
     * Load password rules before page is loaded
     */
    beforeMount() {
        this.getPasswordRules()
    }
   

}
</script>
