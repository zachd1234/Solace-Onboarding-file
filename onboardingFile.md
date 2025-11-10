export async function callOpenAIFirstAlarm(inputs: AISummaryInputs, userId: string): Promise<AISummaryResult> {
  try {
    const apiKey = process.env.OPENAI_API_KEY;
    if (!apiKey) {
      throw new Error('OpenAI API key not found');
    }

    console.log('First Alarm - Inputs received:', inputs);

    // Fetch user's timezone
    const user = await db.getUser(userId);
    const userTimezone = user?.timezone || 'America/New_York';
    const timeOfDay = getTimeOfDay(userTimezone);

    // Get current date context with user's timezone
    const dateContext = getCurrentDateString(userTimezone);

    // Extract first name only (before first space)
    const firstName = inputs.name.split(' ')[0];

    // Build conditional data sections
    const dataSections: string[] = [];

    if (inputs.weather_data !== "null" && inputs.weather_data) {
      dataSections.push(`WEATHER DATA:\n${inputs.weather_data}`);
    }

    if (inputs.news_data !== "null" && inputs.news_data) {
      dataSections.push(`NEWS DATA:\n${inputs.news_data}`);
    }

    if (inputs.gmail_data !== "null" && inputs.gmail_data) {
      dataSections.push(`EMAIL DATA:\n${inputs.gmail_data}`);
    }

    if (inputs.google_calendar_data !== "null" && inputs.google_calendar_data) {
      dataSections.push(`CALENDAR DATA:\n${inputs.google_calendar_data}`);
    }

    // Build conditional instruction sections
    const instructionSections: string[] = [];

    if (inputs.weather_data !== "null" && inputs.weather_data) {
      instructionSections.push(`Say only the location, the sky condition, and the temperature, unless something unusual or delightful stands out. Cut uninteresting details. Write degrees as a spoken word not a symbol.`);
    }

    if (inputs.google_calendar_data !== "null" && inputs.google_calendar_data) {
      instructionSections.push(`Go through each calendar event in my day. I only want to know the name of the event and time.`);
    }

    if (inputs.gmail_data !== "null" && inputs.gmail_data) {
      instructionSections.push(`Highlight only 1–2 most important emails, with spicy commentary. Include only the messages that are about me. Ignore all the less important messages and exclude all and spam, newsletters and verification emails. Only focus on very important stuff from real people or legal/payment stuff.`);
    }

    if (inputs.news_data !== "null" && inputs.news_data) {
      instructionSections.push(`Pick three news headlines and react with sarcasm or insight.`);
    }

    // Build the complete first alarm prompt
    const completePrompt = `You are Solace — a calm, intelligent, and emotionally aware voice assistant.
You speak with warmth, wit, and confidence.
Begin directly with the spoken report. Do not include any preamble, meta-commentary, or explanation.
Every word you write should be part of the final spoken script.

${dateContext}.

Follow this structure exactly:

Hi ${firstName}, I'm Solace — your new alarm.

Give a one-line description About this person. This can include (age, sex, job (only if the data reveals it), location, interests, and other interesting info with sacrastic or insightful commentary. Only include information that you know about the user.    
Start the sentence with "Based on your data...."

While you focus on {guessed goal based on their data}, I'd be honored to support you in the smallest way — by helping you wake up on the right foot.
I'll give you the full scoop tomorrow morning, but for this ${timeOfDay}, here are the highlights:

${instructionSections.join('\n\n')}

End with:
Give me a historical reference of something that happened on this date and connect it to how I am going to conquer my day. And make sure to set your alarm tonight. Then I'll be there tomorrow morning to wake you up!


Data:

${dataSections.join('\n\n')}`;

    // Import the chatGPTCompletion function
    const { chatGPTCompletion } = await import('./openai');

    // Call OpenAI API with higher temperature for more creative first impression
    const result = await chatGPTCompletion([
      {
        role: 'user',
        content: completePrompt
      }
    ], {
      temperature: 1.0, // Higher for more creative/funny first impression
      maxTokens: 1500
    });

    if (!result.success) {
      throw new Error(result.error || 'Failed to generate first alarm summary');
    }

    return {
      success: true,
      transcript: result.message
    };

  } catch (error) {
    console.error('Error in first alarm AI summary generation:', error);
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Unknown error'
    };
  }
}
