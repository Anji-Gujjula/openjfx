String[] stringArray = {"Message 1", "Message 2", "Message 3"};
ArrayList<String> messagesList = new ArrayList<>(Arrays.asList(message));

@FXML
public void handleOptionSelected(ActionEvent event) {
    Button clickedButton = (Button) event.getSource();
    String buttonText = clickedButton.getText();
    ColorOption selectedColor = colorComboBox.getSelectionModel().getSelectedItem();
    
    if (selectedColor == null) {
        showError("Please select a color first");
        return;
    }
    
    // Clear initial state if needed
    if (isInitialState) {
        clearInitialState();
    }
    
    // Check if this button text already exists in the textarea
    boolean textExists = isTextInTextarea(buttonText);
    
    if (textExists) {
        // Remove the text (second click)
        removeTextFromTextarea(buttonText);
        // Update result label to show removal
        resultLabel.setText("Removed: " + buttonText);
        resultLabel.setStyle("-fx-text-fill: red; -fx-font-weight: bold;");
    } else {
        // Add the text (first click)
        addTextToTextarea(buttonText, selectedColor);
        // Update result label to show addition
        resultLabel.setText("Added: " + buttonText);
        resultLabel.setStyle("-fx-text-fill: " + selectedColor.getColorCode() + "; -fx-font-weight: bold;");
        
        // Send to backend only when adding
        sendToBackend(buttonText);
    }
    
    // Update TextFlow display
    updateUnifiedDisplay();
    
    // Clear any errors
    clearError();
}

/**
 * Check if the button text already exists in the textarea
 */
private boolean isTextInTextarea(String buttonText) {
    String currentText = unifiedInputArea.getText();
    if (currentText.isEmpty()) {
        return false;
    }
    
    String[] lines = currentText.split("\n");
    for (String line : lines) {
        if (line.trim().equals(buttonText.trim())) {
            return true;
        }
    }
    return false;
}

/**
 * Add text to textarea with validation
 */
private void addTextToTextarea(String buttonText, ColorOption selectedColor) {
    String currentText = unifiedInputArea.getText();
    String[] currentLines = currentText.isEmpty() ? new String[0] : currentText.split("\n", -1);
    
    // Check current line count
    if (currentLines.length >= MAX_LINES) {
        showError("Can't add more than " + MAX_LINES + " lines");
        return;
    }
    
    // Check character limit
    if (buttonText.length() > MAX_CHARS_PER_LINE) {
        showError("Button text exceeds " + MAX_CHARS_PER_LINE + " character limit");
        return;
    }
    
    // Add button text to unified area
    if (currentText.isEmpty()) {
        unifiedInputArea.setText(buttonText);
    } else {
        unifiedInputArea.appendText("\n" + buttonText);
    }
    
    // Store line with its color for display purposes
    MessageLine newLine = new MessageLine(buttonText, selectedColor, messageLines.size());
    messageLines.add(newLine);
    
    // Send to control manager
    this.controlManager.gotoMessageDisplay(buttonText, selectedColor.getColorCode());
}

/**
 * Remove text from textarea
 */
private void removeTextFromTextarea(String buttonText) {
    String currentText = unifiedInputArea.getText();
    if (currentText.isEmpty()) {
        return;
    }
    
    String[] lines = currentText.split("\n");
    StringBuilder newText = new StringBuilder();
    boolean first = true;
    
    // Rebuild text without the matching line
    for (String line : lines) {
        if (!line.trim().equals(buttonText.trim())) {
            if (!first) {
                newText.append("\n");
            }
            newText.append(line);
            first = false;
        }
    }
    
    // Update the textarea
    unifiedInputArea.setText(newText.toString());
    
    // Remove from messageLines collection
    messageLines.removeIf(messageLine -> 
        messageLine.getText().trim().equals(buttonText.trim()));
    
    // Update line numbers for remaining items
    for (int i = 0; i < messageLines.size(); i++) {
        messageLines.get(i).setLineNumber(i);
    }
}

/**
 * Send data to backend
 */
private void sendToBackend(String buttonText) {
    try {
        EJBServiceLocator serviceLocator = EJBServiceLocator.getInstance();
        DailyFileSLBeanRemote dailyFileSLBeanRemote = serviceLocator.retrieveDailyFileSLBeanEJB();
        UserBean userBean = AVSClient.getInstance().getControlManager().getUserBean();
        dailyFileSLBeanRemote.postDisplayMessage(buttonText, userBean.getLoginId());
        
        //logger.info("Successfully sent message to backend: " + buttonText);
        
    } catch (NamingException ex) {
        logger.severe("Failed to get EJB connection");
        logger.log(Level.SEVERE, ex.getMessage(), ex);
        showError("Failed to send message to database");
    } catch (Exception ex) {
        logger.log(Level.SEVERE, ex.getMessage(), ex);
        showError("Error sending message to backend");
    }
}
