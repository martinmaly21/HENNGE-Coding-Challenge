<script setup lang="ts">
import { ref, watchEffect, onMounted, onUnmounted } from 'vue';

const props = defineProps<{
    recipients: string[]
}>()

//will store reference to the HTML div containing the Email Audit entry data
const container = ref<HTMLElement | null>(null);
// a reusable canvas and context object that will be used for measuring text
let reusableCanvas: HTMLCanvasElement | null = null;
let reusableContext: CanvasRenderingContext2D | null = null;
// Will store the string value of recipients displayed in UI for a given Email Audit entry
const displayedRecipients = ref('');
// Will store the integer value of the number of hidden recipients for a given Email Audit entry
const badgeNumber = ref(0);

//method for initializing canvas that will be used for measuring UI elements
function initializeCanvasIfNeeded() {
    if (!reusableCanvas) {
        reusableCanvas = document.createElement('canvas');
    }

    if (!reusableContext) {
        reusableContext = reusableCanvas.getContext('2d');
    }
}

//This method is responsible for calculating the width for a HTML element when it's populated with a given string. It takes in the string and the element ID, and returns the width in pixels.
const calculateWidthOfTextForElement = (text: string, elementID: string) => {
    let widthOfText = 0
    //get reference to UI element
    const measureElement = document.getElementById(elementID);

    initializeCanvasIfNeeded(); // Ensure canvas is initialized

    if (reusableContext && measureElement) {
        const style = window.getComputedStyle(measureElement, null)
        const fontSizePropertyValue = style.getPropertyValue('font-size');
        const fontFamilyPropertyValue = style.getPropertyValue('font-family');
        const fontSize = parseFloat(fontSizePropertyValue); 
        const fontFamily = fontFamilyPropertyValue.split(', ')[0]
        
        reusableContext.font = `${fontSize}px ${fontFamily}`; // Set the font to the desired value
        const metrics = reusableContext.measureText(text);
        widthOfText = metrics.width;
    } else {
         //if the context could not be retrieved, fallback to a basic calculation where we assume the width of each character to be 8px and add an extra 20px of space for extra allowance
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
    let totalAvailableContainerWidth = (container.value?.clientWidth || 0);

    let recipientsString = '';
    let widthOfEllipses = calculateWidthOfTextForElement(", ...", 'recipients-text-measure-element');

    badgeNumber.value = 0;
    for (const [index, recipient] of props.recipients.entries()) {
        const numberOfHiddenRecipients = (props.recipients.length) - index
        const isFirstRecipient = index === 0;
        const badgeNumberValue = isFirstRecipient ?  numberOfHiddenRecipients - 1 :  numberOfHiddenRecipients;
        const badgeLeftPadding = 2;
        const badgeRightPadding = 2;
        const badgeMargin = 5;
        let widthOfBadge = calculateWidthOfTextForElement(`+${numberOfHiddenRecipients.toString()}`, 'badge-measure-element') + badgeMargin + badgeLeftPadding + badgeRightPadding;

        const isLastRecipient = index === props.recipients.length - 1
        //add a comma and space to all array elements other than the last one
        const recipientText = isLastRecipient ? recipient : `${recipient}, `;
        const widthOfRecipient = calculateWidthOfTextForElement(recipientText, 'recipients-text-measure-element');

        //check if there's enough space to display entire recipient email (+ '...' if there will be a recipient following it )
        if (totalAvailableContainerWidth > widthOfRecipient + (isLastRecipient ? 0 : widthOfEllipses) + (badgeNumberValue > 0 ? widthOfBadge : 0)) {
            recipientsString += recipientText;
            totalAvailableContainerWidth -= ((widthOfRecipient));
        } else {
            recipientsString += isFirstRecipient ? recipient : "...";
            //if there is not enough space to show even the first recipient, we subtract 1 to exclude the first recipient from the badge
            
            badgeNumber.value = badgeNumberValue;
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
    <span class="recipients-text" id="recipients-text-measure-element">{{ displayedRecipients }}</span>
    <span v-if="badgeNumber > 0" class="badge" id="badge-measure-element">+{{ badgeNumber }}</span>
 </div>
</template>

<style scoped>
    .recipients-container {
        display: flex;
        justify-content: space-between;
        align-items: center;
        background-color: red;
    }

    .recipients-text {
        font-size: 16px;
        color: #333333;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        display: block;
    }

    .badge {
        font-size: 16px;
        color: #f0f0f0;
        background-color: #666666;
        border-radius: 3px;
        padding: 2px 5px;
        margin-left: 5px; /* add spacing between the badge and the recipient emails */
        flex: none;
        display: block;
    }
</style>