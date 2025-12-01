<script>
    import { Modal, Hr, Label, Input, Card, Textarea, Select, Timepicker, Button, P, List, Li, Radio, Toggle} from "flowbite-svelte";
    import { CloseOutline, EditOutline, ExclamationCircleOutline, ForwardStepOutline, TrashBinSolid, FileCloneSolid } from "flowbite-svelte-icons";
    import { onMount } from 'svelte';
    import { slide } from 'svelte/transition';


    // Create event vars
    let selectedDate;
    let selectedMonth;
    let selectedYear;

    let title = "";
    let description = "";

    let startTime = 0; //local time
    let endTime = 0; //local time


    onMount(() => {
        const dateObj = new Date();
        selectedDate = dateObj.getDate();
        selectedMonth = dateObj.getMonth()+1; // Months are 0-indexed in JS Date
        selectedYear = dateObj.getFullYear();
        isQuickMode = false;
    });

    // Months for the dropdown
    let months = [
        { value: 1, name: "January" },
        { value: 2, name: "February" },
        { value: 3, name: "March" },
        { value: 4, name: "April" },
        { value: 5, name: "May" },
        { value: 6, name: "June" },
        { value: 7, name: "July" },
        { value: 8, name: "August" },
        { value: 9, name: "September" },
        { value: 10, name: "October" },
        { value: 11, name: "November" },
        { value: 12, name: "December" }
    ];

    let events = []; // Array to hold created events

    //
    function padNum(num){
        return num.toString().padStart(2,'0');
    }


    function toUTCISOString(year, month, day, time){
        const [hours, minutes] = time.split(':').map(Number);
        const date = new Date(year, month -1, day, hours, minutes);
        const YYYY = date.getUTCFullYear();
        const MM = padNum(date.getUTCMonth() + 1); // Months are 0-indexed
        const DD = padNum(date.getUTCDate());
        const hh = padNum(date.getUTCHours());
        const mm = padNum(date.getUTCMinutes());
        const ss = "00"; // Seconds are always 00 in this case
        return `${YYYY}${MM}${DD}T${hh}${mm}${ss}Z`;
    }

    // Function to handle event creation
    function createEvent(){

        // error handling: ensure all fields are filled (besides description)
        if (!title) {
            alert("Please enter a valid event title!");
            return;
        }

        //create event object
        const newEvent = {
            title: title,
            description: description,
            month:selectedMonth,
            date:selectedDate,
            year:selectedYear,
            startTime: startTime, //24 hr format
            endTime: endTime, //24 hr format
            // // UTC time format: YYYYMMDDTHHmmssZ
            start: toUTCISOString(selectedYear, selectedMonth, selectedDate, startTime),
            end: toUTCISOString(selectedYear, selectedMonth, selectedDate, endTime)
            // start: selectedYear + padNum(selectedMonth) + padNum(selectedDate) + 'T'+ toISOString(startTime.split(':')[0]) + toISOString(startTime.split(':')[1]) +'00Z',
            // end: selectedYear + padNum(selectedMonth) + padNum(selectedDate) + 'T' + toISOString(endTime.split(':')[0]) +  toISOString(endTime.split(':')[1]) + '00Z'
        }
        events = [...events, newEvent];

        // DEBUGGING:
        events.forEach(event => {
            console.log(event.title + ' ' + event.description + ' ' + event.start + ' ' + event.end);
        });

        // clear form fields
        title = "";
        description = "";
        startTime = "";
        endTime = "";
        selectedDate = new Date().getDate();
        selectedMonth = new Date().getMonth() + 1;
        selectedYear = new Date().getFullYear();
    }
    

    // Delete event modal
    let delete_popupModal = false;
    let event_to_delete;

    // Edit event modal
    let edit_popupModal = false;
    let error = "";
    let event_to_edit;

    // Convert military time (24hr) to regular time (12hr)
        //Purpose: for displaying created events
    function toRegTime(militaryTime){

        //format: hh:mm
        let hrs = parseInt(militaryTime.split(':')[0], 10);

        const AmPM = hrs >= 12 ? 'PM' : 'AM'; 
        
        hrs = hrs > 12 ? hrs-12 : hrs; // convert to 12 hr format
        if (hrs === 0) hrs = 12; // 00 hrs is 12 AM

        const mins = militaryTime.split(':')[1];
      
        return (hrs + ':' + mins + ' ' + AmPM);
    }

    function toMilitaryTime(hr, isAM){
        hr = Number(hr);
        if (isAM === "am") //12 AM -> 00
            return hr===12? padNum(0) : padNum(hr);
        else //12 PM -> 12
            return hr===12? padNum(12) : padNum(hr+12);
    }


    // Function to generate and download .ics file
    function addToCalendar(){
          const now = new Date()
            .toISOString()
            .replace(/[-:]/g, '')
            .split('.')[0];  // 20250929T180000 format
            
        //turn all the events into a single string
        const vevents = events.map(event => `BEGIN:VEVENT
UID:${crypto.randomUUID()}@myapp
DTSTAMP:${now}
SUMMARY:${event.title}
DESCRIPTION:${event.description}
DTSTART:${event.start}
DTEND:${event.end}
END:VEVENT`).join('\r\n');

        const ics = `BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//MyApp//EN
${vevents}
END:VCALENDAR`;

        const blob = new Blob([ics], {type:"text/calendar"});
        const url = URL.createObjectURL(blob);

        const link = document.createElement('a');
        link.href = url;
        link.download = 'events.ics';
        link.style.display = 'none'; //keep it hidden
        document.body.appendChild(link); //must be in DOM for Safari
        link.click();
        document.body.removeChild(link); // clean up
        URL.revokeObjectURL(url);
    }

    // quick mode
    let isQuickMode = false;
    let dateText;
    let startTimeText;
    let endTimeText;
    let isStartAM = true;
    let isEndAM = true;
    let selectedStartAM = "am";
    let selectedEndAM = "am";


    //helper function that parse out the month, day, and year from MM/DD/YYYY format
    function parseDate()
    {
        // dataText format : mm/dd/yyyy
        const [mm, dd, yyyy] = dateText.split('/').map(x=>x.trim());
        selectedMonth = Number(mm);
        selectedDate = dd;
        selectedYear = yyyy;
    }

    //helper fucntion parse out the start and end times and ensure it is in 24 hr HH:MM format
    function parseTime()
    {
        const [start_hh, start_mm] = startTimeText.split(':').map(x => x.trim());
        console.log(start_hh + " " + start_mm);
        const [end_hh, end_mm] = endTimeText.split(':').map(x => x.trim());
        console.log(end_hh + " " + end_mm);

        startTime = toMilitaryTime(start_hh, selectedStartAM) + ':' + start_mm;
        console.log("st: " + startTime);
        endTime = toMilitaryTime(end_hh, selectedEndAM) + ':' + end_mm;
        console.log("et: " + endTime);
    }

    //function that grabs all the text data and create event when in quick mode
    function createQuickEvent(){
        if (title != '') {
            parseDate();
            parseTime();
            createEvent();
            //clear out date, start and end time field and amPM radio buttons
            dateText = '';
            startTimeText = '';
            endTimeText = '';
            isStartAM = true;
            isEndAM = true;
            selectedStartAM = "am";
            selectedEndAM = "am";
        }
        else
            alert("Please enter a valid event title!");
            return;
    }

</script>

<div class="flex flex-col items-center max-w-full overflow-x-hidden">

    <div class="flex flex-col items-center m-5 mb-10">
        <!-- Title -->
        <h1 class="text-center uppercase font-bold text-3xl mb-3">Apple Calendar Event Tool</h1>

        <!-- Description -->
        <p class="text-justify text-gray-700 w-5/6 sm:w-2/5 text-based">A simple tool to create and download Apple Calendar (.ics) event files. Perfect for adding events to your calendar quickly and easily.</p>
    </div>


  
    <div class="w-full">

        <!-- QUICK MODE -->
        {#if isQuickMode}
            <form class=" flex flex-col items-center pb-15">

            
                <!-- Quick mode toggle -->
                <div class="ml-50 mb-2">
                    <Toggle bind:checked={isQuickMode}><span class="italic mr-1">Quick </span> Mode</Toggle>
                </div>

                <Card class="p-4 sm:p-6 md:p-8 w-5/6 rounded-2xl shadow-2xl ">
                    
                    <h5 class="mb-2 text-2xl font-bold tracking-tight text-gray-900 dark:text-white uppercase text-center">Quick Mode</h5>

                    <p class="text-center">Type in all the text boxes. Be sure to follow the format!</p>

                    <Hr class="mx-auto my-3 h-1 w-48 rounded-sm md:my-5"/>

                    <h1 class="uppercase text-sm font-bold">Title</h1>
                    <Input placeholder="Event TITLE..." bind:value={title}/>

                    <h1 class="mt-2 uppercase text-sm font-bold">Description</h1>
                    <Textarea
                            id="description-textarea" 
                            placeholder="Event DESCRIPTION..."
                            rows={2}
                            name="description"
                            class="w-full mb-2"
                            bind:value={description}
                    />

                    <h1 class="uppercase text-sm font-bold">Date</h1>
                    <!-- Must parse out data -->
                    <Input bind:value={dateText} placeholder="MM/DD/YYYY"/>


                    <div class="flex justify-between mt-2"> 
                        <h1 class="mr-2 my-auto uppercase text-sm font-bold">Start time</h1>

                        <!-- Must write function to parse out value from text -->
                        <Input bind:value={startTimeText} placeholder="HH:MM" class="w-2/3 sm:w-1/2 my-2 mx-2"/> 

                        <div class="w-1/2 flex justify-between">
                            <Radio name="startTimeAMPM" value="am" bind:group={selectedStartAM}>A.M.</Radio>
                            <Radio name="startTimeAMPM" value="pm" bind:group={selectedStartAM}>P.M.</Radio>
                        </div>
                        
                    </div>

                    <div class="flex justify-between"> 
                        <h1 class="mr-4 my-auto uppercase text-sm font-bold">End time</h1>

                        <!-- Must write function to parse out value from text -->
                        <Input bind:value={endTimeText} placeholder="HH:MM" class="w-2/3 sm:w-1/2 my-2 mx-2"/> 

                        <div class="w-1/2 flex justify-between">
                            <Radio name="endTimeAMPM" value="am" bind:group={selectedEndAM}>A.M.</Radio>
                            <Radio name="endTimeAMPM" value="pm" bind:group={selectedEndAM}>P.M.</Radio>
                        </div>
                        
                    </div>

                    
                    <!-- New function to parse out start and end time -->
                    <Button class="w-fit mx-auto mt-5" onclick={createQuickEvent}>Add Event</Button>
                </Card>
            </form>
        {:else}
            <!-- Event Form: NORMAL MODE -->
            <form class="flex flex-col items-center pb-15">

                <!-- Quick mode toggle -->
                <div class="ml-50 mb-2">
                    <Toggle bind:checked={isQuickMode}><span class="italic mr-1">Quick </span> Mode</Toggle>
                </div>
                
                <Card class="p-4 sm:p-6 md:p-8 w-5/6 rounded-3xl shadow-2xl">
        
                    <!-- Event Header -->
                    <div class="text-center">
                        <h5 class="mb-2 text-2xl font-bold tracking-tight text-gray-900 dark:text-white uppercase">New Event</h5>
                        <p class="leading-tight font-normal text-gray-700 dark:text-gray-400">Start here by creating a new event!</p>
                    </div>
                    
                    <Hr class="mx-auto my-3 h-1 w-48 rounded-sm md:my-5"/>
                    
                    <!-- Event Title -->
                    <div class="mb-6">
                        <Label for="title-input" class="mb-2 block">Enter Event <p class="uppercase font-bold inline-block text-[16px]">Title</p></Label>
                        <Input id="title-input" placeholder="Title..." bind:value={title}/>
                    </div>
            
                    <!-- Event Description -->
                    <div class="mb-6">
                        <Label for="description-textarea" class="mb-2 block">Enter Event <p class="uppercase font-bold inline-block text-[16px]">Description</p></Label>
                        <Textarea
                            id="description-textarea" 
                            placeholder="Description..."
                            rows={2}
                            name="description"
                            class="w-full"
                            bind:value={description}
                        />
                    </div>
        
                    <Hr class="mx-auto my-3 h-1 w-48 rounded-sm md:my-5"/>
        
                    <!-- Event Time -->
                    <div class="mb-6">
                        <Label for="time-input" class="mb-2 block">Select Event <p class="uppercase font-bold inline-block text-[16px]">Time</p></Label>
        
                        <!-- DATE -->
                        <div>
                            <!-- MONTH -->
                            <Select class="inline-block" items={months} bind:value={selectedMonth} />
            
                            <!-- DAY -->
                            <Input type="number" min="1" max="31" placeholder={selectedDate} class="inline-block w-auto" bind:value={selectedDate}/>
            
                            <!-- YEAR -->
                            <Input type="number" min="2025" max="2100" placeholder={selectedYear} bind:value={selectedYear} class="w-auto inline-block"/>
                        </div>
        
        
                        <!-- TIME -->
                        <div class="mt-4  flex justify-between">
                            <div class="inline-block mr-4 ">
                                <!-- START -->
                                <Label for="start-time">Select <p class="uppercase font-bold inline-block text-[16px]">Start</p> Time</Label>
                                <Timepicker id="start-time" divClass="w-full" bind:value={startTime}/>
                            </div>
        
                            <div class="inline-block ">
                                <!-- END -->
                                <Label for="end-time">Select <p class="uppercase font-bold inline-block text-[16px]">End</p> Time</Label>
                                <Timepicker id="end-time" divClass="w-full" bind:value={endTime}/>
                            </div>
                        </div>
                    
                    </div>
        
        
                    <!-- Submit Button -->
                    <div class="text-center">
                        <Button class="w-36" onclick={createEvent}> Create Event </Button>
                    </div>
                </Card>
        
            </form>
        {/if}
    </div>
   
            

    <!-- List of Created Events -->
    {#if events.length > 0}
        <Card class="p-4 m-5 sm:p-6 md:p-8 w-5/6 bg-[#CDD1C7]">
            <h1 class="text-center uppercase font-bold text-2xl">Created Event(s)</h1>

            <!-- List all the create events -->
            <List>
                {#each events as event}
                    <div class="m-2">

                        <!-- Event title and full date -->
                        <div class="flex justify-between">
                            <h1 class="font-bold inline-block">{event.title}</h1>
                            <h1 class="inline-block font-bold">{event.month + "/" + event.date + "/" + event.year}</h1>
                        </div>

                        <!-- Event description and times  -->
                        <div class="flex justify-between">
                            <p class="italic">{event.description}</p>
                            <h1 class="inline-block">{toRegTime(event.startTime) + ' - ' + toRegTime(event.endTime)}</h1>
                        </div>

                        <!-- Edit, delete, duplicate btns -->
                        <div class="text-right mt-1">

                            <!-- Duplicate current event btn -->
                            <Button size="sm" class="h-5 p-3" onclick={() => {
                                //create a new event with same values
                                const duplicateEvent = {
                                    // title: event.title,
                                    // description: event.description,
                                    // month:event.selectedMonth,
                                    // date:event.selectedDate,
                                    // year:event.selectedYear,
                                    // startTime: event.startTime,
                                    // endTime: event.endTime,

                                    //cleaner way
                                    ...event,

                                    // UTC time format: YYYYMMDDTHHmmssZ
                                    start: toUTCISOString(event.selectedYear, event.selectedMonth, event.selectedDate, event.startTime),
                                    end: toUTCISOString(event.selectedYear, event.selectedMonth, event.selectedDate, event.endTime)
                                }
                                
                                //add duplicate event into event list
                                events = [...events, duplicateEvent];
                            }}>
                                <FileCloneSolid/>
                            </Button>
                        

                            <!-- Edit btn -->
                            <Button size="sm" class="h-5 p-3" onclick={() => {
                                    // Open edit event modal
                                    edit_popupModal = true;
                                    event_to_edit = event;
                                }}>
                                <EditOutline/>
                            </Button>
                            
                            <!-- Delete btn -->
                            <Button size="sm" class="h-5 p-3" onclick={() => {
                                // Confirm delete event msg
                                delete_popupModal = true;
                                event_to_delete = event;
                            }}> 
                                <TrashBinSolid/>
                            </Button>

                        </div>
                    </div>
                    <hr>
                {/each}
            </List>
        
            <Button class="mt-10 w-36 mx-auto" onclick={addToCalendar}>Add to Calendar</Button>
    
        </Card>
    {/if}

    <!-- Delete event modal -->
    <Modal form bind:open={delete_popupModal} class="w-5/6 sm:w-auto" size="xs" transition={slide} permanent>
        <div class="text-center">
            <ExclamationCircleOutline class="mx-auto mb-4 h-12 w-12 text-gray-400 dark:text-gray-200" />
            <h3 class="mb-5 text-lg font-normal text-gray-500 dark:text-gray-400">Are you sure you want to delete this event?</h3>
            <div class="space-x-2">
            <Button type="submit" value="yes" color="red" onclick={() => {
                // Remove event from events array
                events = events.filter(currEvent => currEvent !== event_to_delete);
                delete_popupModal = false;
            }}>Yes, delete</Button>
            <Button type="submit" value="no" color="alternative">No, cancel</Button>
            </div>
        </div>
    </Modal>

    <!-- Edit event modal -->
    <Modal form bind:open={edit_popupModal} class="w-5/6 sm:w-auto" size="xs" transition={slide} permanent>
        <div class="flex flex-col space-y-6">
            <h3 class="mb-4 text-2xl font-bold uppercase text-gray-900 dark:text-white text-center">Edit Event</h3>
            
            <Hr class="mx-auto my-3 h-1 w-48 rounded-sm md:my-5"/>

            <!-- Event title -->
            <Label class="space-y-2">
                <span>Event <p class="uppercase font-bold inline-block text-[16px]">Title</p></span>
                <Input bind:value={event_to_edit.title}/>
            </Label>

            <!-- Event Description -->
            <Label class="space-y-2">
                <span>Event <p class="uppercase font-bold inline-block text-[16px]">Description</p></span>
                <Input bind:value={event_to_edit.description}/>
            </Label>

            <!-- Event Time -->
            <Label class="space-y-2">
                <span>Event <p class="uppercase inline-block font-bold text-[16px]">Time</p></span>

                <!-- DATE -->
                <div>
                    <!-- MONTH -->
                    <Select class="inline-block" items={months} bind:value={event_to_edit.month} />
    
                    <!-- DAY -->
                    <Input type="number" min="1" max="31" bind:value={event_to_edit.date} class="inline-block w-auto"/>
    
                    <!-- YEAR -->
                    <Input type="number" min="2025" max="2100" bind:value={event_to_edit.year} class="w-auto inline-block"/>
                </div>

                <!-- TIME -->
                <div class="flex justify-between mt-5">
                    <!-- START -->
                    <div class="inline-block">
                        <Label for="start-time"><p class="uppercase font-bold inline-block text-[16px]">Start</p> Time</Label>
                        <Timepicker id="start-time" divClass="inline-block" bind:value={event_to_edit.startTime}/>
                    </div>

                    <!-- END -->
                    <div class="inline-block">
                        <Label for="end-time"><p class="uppercase font-bold inline-block text-[16px]">End</p> Time</Label>
                        <Timepicker id="end-time" divClass="inline-block" bind:value={event_to_edit.endTime}/>
                    </div>

                </div>
            </Label>
    
            <Hr class="mx-auto my-3 h-1 w-48 rounded-sm md:my-5"/>

            <Button type="submit" size="sm" class="w-fit mx-auto mt-5" onclick={()=>{

                event_to_edit.start = toUTCISOString(event_to_edit.year, event_to_edit.month, event_to_edit.date, event_to_edit.startTime);
                event_to_edit.end = toUTCISOString(event_to_edit.year, event_to_edit.month, event_to_edit.date, event_to_edit.endTime);

                // // re-add updated event
                events = events;
          
                // close modal
                edit_popupModal = false;

                // DEBUGGING:
                events.forEach(event => {
                    console.log(event.title + ' ' + event.description + ' ' + event.start + ' ' + event.end);
                });
            }}>
                Confirm Edit
            </Button>

        </div>
    </Modal>


</div>

