<script>
    import { Modal, Hr, Label, Input, Card, Textarea, Select, Timepicker, Button, P, List, Li, Radio} from "flowbite-svelte";
    import { CloseOutline, EditOutline, ExclamationCircleOutline, ForwardStepOutline } from "flowbite-svelte-icons";
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
            alert("Please fill in all required fields.");
            return;
        }

        //create event object
        const newEvent = {
            title: title,
            description: description,
            month:selectedMonth,
            date:selectedDate,
            year:selectedYear,
            startTime: startTime,
            endTime: endTime,
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
    function toRegTime(militaryTime){

        //format: hh:mm
        let hrs = parseInt(militaryTime.split(':')[0], 10);

        const AmPM = hrs >= 12 ? 'PM' : 'AM'; 
        
        hrs = hrs > 12 ? hrs-12 : hrs; // convert to 12 hr format
        if (hrs === 0) hrs = 12; // 00 hrs is 12 AM

        const mins = militaryTime.split(':')[1];
      
        return (hrs + ':' + mins + ' ' + AmPM);
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
</script>

<div class="flex flex-col items-center">

    <div class="flex flex-col items-center m-5">
        <!-- Title -->
        <h1 class="text-center uppercase font-bold text-3xl mb-3">Apple Calendar Event Tool</h1>

        <!-- Description -->
        <p class="text-center text-gray-700">A simple tool to create and download Apple Calendar (.ics) event files. Perfect for adding events to your calendar quickly and easily.</p>
    </div>


        <!-- FUTURE IMPLEMENTATION: Quick mode -->
    {#if isQuickMode}
        <form class="w-full flex flex-col items-center">
            <Card class="p-4 sm:p-6 md:p-8 w-5/6">
                <p class="text-sm italic text-right m-2">Try NORMAL Mode
                    <Button size="sm" class="w-4 h-5" onclick={()=>{isQuickMode=!isQuickMode}}> <ForwardStepOutline class="shrink-0 h-4 w-4 inline-block" /></Button> 
                </p>
                <h1 class="text-center font-bold uppercase p-2">Quick Mode</h1>

                <Input placeholder="Event TITLE..." bind:value={title}/>

                <Textarea
                        id="description-textarea" 
                        placeholder="Event DESCRIPTION..."
                        rows={2}
                        name="description"
                        class="w-full my-2"
                        bind:value={description}
                />
                <Input placeholder="MM/DD/YYYY"/>

                <div class="flex justify-between">
                    <!-- Must write function to parse out value from text -->
                    <Input placeholder="START time [HH:MM]" class="w-fit my-2"/> 
                    <Radio name="startTimeAMPM" value="am">A.M.</Radio>
                    <Radio name="startTimeAMPM" value="pm">P.M.</Radio>
                </div>

                <div class="flex justify-between">
                    <!-- Must write function to parse out value from text -->
                    <Input placeholder="END time [HH:MM]" class="w-fit"/>
                    <Radio name="endTimeAMPM" value="am">A.M.</Radio>
                    <Radio name="endTimeAMPM" value="pm">P.M.</Radio>
                </div>
                <!-- New function to parse out start and end time -->
                <Button class="w-fit mx-auto mt-5">Add Event</Button>
            </Card>
        </form>
        {:else}
            <!-- Event Form: Normal mode -->
            <form class="w-full flex flex-col items-center">
        
                <Card class="p-4 sm:p-6 md:p-8 w-5/6">
        
                    <p class="text-sm italic text-right m-2">Try QUICK Mode 
                        <Button size="sm" class="w-4 h-5" onclick={()=>{isQuickMode=!isQuickMode}}> <ForwardStepOutline class="shrink-0 h-4 w-4 inline-block" /></Button> 
                    </p>
        
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
            

    <!-- List of Created Events -->
    {#if events.length > 0}
        <Card class="p-4 m-5" size="lg">
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

                        <!-- Edit and delete btns -->
                        <div class="text-right mt-1">

                            <!-- Delete btn -->
                            <Button size="sm" class="h-5 p-3" onclick={() => {
                                // Confirm delete event msg
                                delete_popupModal = true;
                                event_to_delete = event;
                            }}> 
                                <CloseOutline/>
                            </Button>

                            <!-- Edit btn -->
                            <Button size="sm" class="h-5 p-3" onclick={() => {
                                // Open edit event modal
                                edit_popupModal = true;
                                event_to_edit = event;
                            }}>
                                <EditOutline/>
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

