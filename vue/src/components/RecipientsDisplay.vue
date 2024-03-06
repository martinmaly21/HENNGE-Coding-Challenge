<script setup lang="ts">
import { ref, watchEffect, onMounted, onUnmounted } from 'vue';

const props = defineProps<{
    recipients: string[]
}>()

//will store reference to the HTML div containing the Email Audit entry data
const container = ref<HTMLElement | null>(null);
// Will store the string value of recipients displayed in UI for a given Email Audit entry
const displayedRecipients = ref('');
// Will store the integer value of the number of hidden recipients for a given Email Audit entry
const badgeNumber = ref(0);

//This method is responsible for calculating the width of a string UI. It takes in the string, and returns the width in pixels.
const calculateWidthOfText = (text: string) => {
    let widthOfText = 0
    //get reference to hidden UI element
    const measureElement = document.getElementById('measure-element');
        if (measureElement) {
            //update the text of the HTML element, and force a layout pass
            measureElement.textContent = text;
            //measure the width of that HTML element populated with the text
            widthOfText = measureElement.offsetWidth;
        } else {
            //if the HTML element could not be retrieved, fallback to a basic calculation where we assume the width of each character to be 8px and add an extra 20px of space for extra allowance
            widthOfText = text.length * 8 + 20
        }         
    return widthOfText
};

/*
This method uses the current width of the RecipientsDisplay container to conditionally update the values of 'displayedRecipients' and 'badgeNumber' such that the UI follows the following specs:
- If all the recipient email addresses fit in the available space, we can simply display them delimited by a comma and space (e.g. `a, b`).
- When there is not enough space to display all recipients, we trim the text. However, to prevent showing clipped email addresses that are hard to read, we trim entire email addresses. If we cannot fit the entirety of a recipient email address, it shouldn't be shown at all.
- When at least one recipient is trimmed, we put a comma, space, and ellipsis after the last fitting recipient (e.g. `a, b, ...`). Furthermore, the rightmost end of the column should show a "badge" with the number of trimmed recipients (`+N`).
- If there is not enough space to show even the first recipient, the badge should show the number of trimmed recipients excluding the first recipient, and the recipient should be truncated with an ellipsis only. If there is only one recipient, there will be no badge, and the recipient should still be truncated by an ellipsis.
*/
const formatRecipientsForDisplay = () => {
    //stores the current width of the RecipientsDisplay container
    const leftContainerPadding = 0; // i think i can remove
    const rightContainerPadding = 0; //i think i can remove
    const containerWidth = (container.value?.clientWidth || 0) - leftContainerPadding - rightContainerPadding;

    const leftBadgeContainerPadding = 5;
    const rightBadgeContainerPadding = 5;
    const badgeTextWidth = 0; //TODO: how to calculate? 
    const badgeWidth = leftBadgeContainerPadding + rightBadgeContainerPadding + badgeTextWidth;

    var totalAvailableContainerWidth = containerWidth - leftContainerPadding - rightContainerPadding;


    let currentWidth = 0;
    let recipientsString = '';

    let widthOfEllipses = calculateWidthOfText("...");

    for (const [index, recipient] of props.recipients.entries()) {
        let recipientText = index === props.recipients.length - 1 ? recipient : `${recipient}, `;
        let widthOfRecipient = calculateWidthOfText(recipientText);

        if (totalAvailableContainerWidth > widthOfRecipient + (index === props.recipients.length - 1 ? 0 : widthOfEllipses)) {
            recipientsString += recipientText;
            totalAvailableContainerWidth -= widthOfRecipient;
        } else {
            recipientsString += "...";
            break; // No more space, so exit the loop
        }
    }

    displayedRecipients.value = recipientsString;
};

//This ensures that the UI will update reactively if the recipient data were to ever change
watchEffect(formatRecipientsForDisplay);

//The following hooks are responsible for updating the UI reactively when the window size changes
onMounted(() => {
  window.addEventListener('resize', formatRecipientsForDisplay);
});

onUnmounted(() => {
  window.removeEventListener('resize', formatRecipientsForDisplay);
});
</script>

<template>
 <div ref="container" class="recipients-container">
    <span class="recipients-text" id="measure-element" style="visibility: hidden; white-space: nowrap;"></span>
    <span class="recipients-text">{{ displayedRecipients }}</span>
    <span v-if="badgeNumber > 0" class="badge">+{{ badgeNumber }}</span>
 </div>
</template>

<style scoped>
    .recipients-container {

    }

    .recipients-text {

    }

    .badge {
        
    }
</style>