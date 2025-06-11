package gov.ca.lc.controllers.admin;

import gov.ca.lc.utilities.constants.ImageConstants;

import java.net.URL;
import java.util.ResourceBundle;
import java.util.ArrayList;
import gov.ca.lc.utilities.EJBServiceLocator;
import javafx.collections.FXCollections;
import javafx.collections.ObservableList;
import javafx.fxml.FXML;
import javafx.fxml.Initializable;
import javafx.scene.Scene;
import javafx.scene.control.ComboBox;
import javafx.scene.control.TextFormatter;
import javafx.scene.input.KeyCode;
import gov.ca.lc.dailyfile.DailyFileSLBeanRemote;
import java.util.logging.Level;
import java.util.logging.Logger;
import gov.ca.lc.controllers.AVSClient;
import javax.naming.NamingException;
import javafx.event.ActionEvent;
import javafx.fxml.FXML;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.scene.control.TextArea;
import javafx.scene.layout.FlowPane;
import javafx.scene.control.ScrollPane;
import javafx.scene.text.TextFlow;
import javafx.scene.text.Text;
import javafx.scene.paint.Paint;

import javafx.animation.PauseTransition;
import javafx.util.Duration;
import javafx.application.Platform;
import java.util.List;
import gov.ca.lc.controllers.AVSControlManager;

public class DisplayMessageController implements Initializable, ImageConstants {
    private static final Logger logger = Logger.getLogger(AdminHomeController.class.getPackage().getName());

    @FXML private FlowPane buttonPane; 
    @FXML private Label resultLabel; 
    @FXML private Label errorLabel;
    @FXML private ComboBox<ColorOption> colorComboBox;
    @FXML private Button clearButton;
    @FXML private Button sendButton;
    @FXML private TextFlow messageDisplayFlow;
    @FXML private ScrollPane messageScrollPane;
    @FXML private TextArea unifiedInputArea;

    // Store each line with its color information
    private List<MessageLine> messageLines;
    private ObservableList<ColorOption> colorOptions;
    private static final int MAX_LINES = 10;
    private static final int MAX_CHARS_PER_LINE = 100;
    private boolean isInitialState = true;
    private AVSControlManager controlManager;

    @Override
    public void initialize(URL url, ResourceBundle rb) {
        logger.info("Initializing DisplayMessageController");
        
        // Initialize message lines storage
        messageLines = new ArrayList<>();
        
        // Initialize components
        initializeColors(); 
        initializeMessageDisplay();
        setupCustomInputValidation();
        
        // Initialize control manager
        this.controlManager = AVSClient.getInstance().getControlManager();
        
        // Load buttons from service
        loadButtonsFromService();
        
        // Initialize error label
        initializeErrorLabel();
    }

    private void initializeErrorLabel() {
        if (errorLabel == null) {
            logger.warning("errorLabel is null - make sure it's defined in FXML and has fx:id='errorLabel'");
        } else {
            errorLabel.setVisible(false);
            errorLabel.setManaged(false);
        }
    }

    private void initializeColors() {
        colorOptions = FXCollections.observableArrayList(
            new ColorOption("Green", 1, "#00CC00"),
            new ColorOption("Red", 2, "#FF0000"),
            new ColorOption("Blue", 3, "#0066FF"),
            new ColorOption("Orange", 4, "#FF8C00"),
            new ColorOption("Yellow", 5, "#FFD700"),
            new ColorOption("Purple", 6, "#9932CC"),
            new ColorOption("Black", 7, "#000000")
        );

        colorComboBox.setItems(colorOptions);
        colorComboBox.getSelectionModel().selectFirst(); // Select first color by default
    }

    private void initializeMessageDisplay() {
        // Set initial placeholder text in TextFlow
        displayInitialPlaceholder();
        isInitialState = true;
        
        // Initialize unified input area
        setupUnifiedInputArea();
    }

    private void setupUnifiedInputArea() {
        unifiedInputArea.clear();
        unifiedInputArea.setPromptText("Click buttons above or type your custom message here...");
        unifiedInputArea.setWrapText(true);
        
        // Add text change listener to update TextFlow display
        unifiedInputArea.textProperty().addListener((observable, oldValue, newValue) -> {
            if (!isInitialState) {
                updateUnifiedDisplay();
            }
        });
        
        // Add focus listener to clear initial state
        unifiedInputArea.focusedProperty().addListener((observable, oldValue, newValue) -> {
            if (newValue && isInitialState) {
                clearInitialState();
            }
        });
    }

    private void displayInitialPlaceholder() {
        messageDisplayFlow.getChildren().clear();
        
        // Create placeholder lines
        for (int i = 0; i < 7; i++) {
            Text placeholderLine = new Text("--------");
            placeholderLine.setFill(Paint.valueOf("#CCCCCC"));
            placeholderLine.setStyle("-fx-font-size: 14px; -fx-font-family: 'Courier New';");
            
            messageDisplayFlow.getChildren().add(placeholderLine);
            
            if (i < 6) {
                Text newLine = new Text("\n");
                messageDisplayFlow.getChildren().add(newLine);
            }
        }
    }

    private void setupCustomInputValidation() {
        // Set up TextFormatter for unified input validation
        unifiedInputArea.setTextFormatter(new TextFormatter<>(change -> {
            if (isInitialState) {
                return change;
            }
            
            if (change.isDeleted()) {
                Platform.runLater(() -> clearError());
                return change;
            }

            String newText = change.getControlNewText();
            if (newText == null || newText.isEmpty()) {
                Platform.runLater(() -> clearError());
                return change;
            }

            String[] lines = newText.split("\n", -1);

            // Check line count limit
            if (lines.length > MAX_LINES) {
                Platform.runLater(() -> showError("Can't enter more than " + MAX_LINES + " lines"));
                return null;
            }

            // Check character limit for each line
            for (String line : lines) {
                if (line.length() > MAX_CHARS_PER_LINE) {
                    Platform.runLater(() -> showError("Can't enter more than " + MAX_CHARS_PER_LINE + " characters per line"));
                    return null;
                }
            }

            Platform.runLater(() -> clearError());
            return change;
        }));

        // Handle Enter key validation
        unifiedInputArea.setOnKeyPressed(event -> {
            if (isInitialState) return;
            
            if (event.getCode() == KeyCode.ENTER) {
                String currentText = unifiedInputArea.getText();
                String[] lines = currentText.split("\n", -1);
                
                if (lines.length >= MAX_LINES) {
                    showError("Can't enter more than " + MAX_LINES + " lines");
                    event.consume();
                } else {
                    clearError();
                }
            }
        });
    }

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

        // Check current line count
        String currentText = unifiedInputArea.getText();
        String[] currentLines = currentText.isEmpty() ? new String[0] : currentText.split("\n", -1);
        
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

        // Update TextFlow display
        updateUnifiedDisplay();
        
        // Update result label
        resultLabel.setText("Added: " + buttonText + " (" + selectedColor.getName() + ")");
        resultLabel.setStyle("-fx-text-fill: " + selectedColor.getColorCode() + "; -fx-font-weight: bold;");
        
        // Clear any errors
        clearError();
        
        // Send to backend
        this.controlManager.gotoMessageDisplay(buttonText);
    }

    @FXML
    public void processCustomText(ActionEvent event) {
        String fullText = unifiedInputArea.getText().trim();
        
        if (fullText.isEmpty()) {
            showError("Please enter some text");
            return;
        }

        ColorOption selectedColor = colorComboBox.getSelectionModel().getSelectedItem();
        if (selectedColor == null) {
            showError("Please select a color first");
            return;
        }

        // Clear initial state if needed
        if (isInitialState) {
            clearInitialState();
        }

        // Split text into lines and process color assignments
        String[] lines = fullText.split("\n");
        messageLines.clear(); // Clear existing stored lines
        
        // Process each line and assign colors
        for (int i = 0; i < lines.length; i++) {
            String line = lines[i].trim();
            if (!line.isEmpty()) {
                // Check if this line was added by a button (preserve button colors)
                // For now, assign current selected color to all custom-typed lines
                MessageLine messageLine = new MessageLine(line, selectedColor, i);
                messageLines.add(messageLine);
            }
        }

        // Update TextFlow display
        updateUnifiedDisplay();
        
        // Show success message
        resultLabel.setText("Text processed successfully! (" + lines.length + " lines in " + selectedColor.getName() + ")");
        resultLabel.setStyle("-fx-text-fill: " + selectedColor.getColorCode() + "; -fx-font-weight: bold;");
        
        // Clear errors
        clearError();
        
        // Send to backend
        this.controlManager.gotoCustomText(lines);
    }

    private void clearInitialState() {
        if (isInitialState) {
            messageLines.clear();
            messageDisplayFlow.getChildren().clear();
            unifiedInputArea.clear();
            isInitialState = false;
        }
    }

    private void updateUnifiedDisplay() {
        // Update the TextFlow display based on current text and stored colors
        updateMessageDisplay();
    }

    private void updateMessageDisplay() {
        // Clear existing content
        messageDisplayFlow.getChildren().clear();
        
        if (isInitialState || unifiedInputArea.getText().trim().isEmpty()) {
            displayInitialPlaceholder();
            return;
        }

        // Get current text from unified input area
        String currentText = unifiedInputArea.getText();
        String[] currentLines = currentText.split("\n");
        
        // Display each line with appropriate color
        ColorOption defaultColor = colorComboBox.getSelectionModel().getSelectedItem();
        if (defaultColor == null) defaultColor = colorOptions.get(0);
        
        for (int i = 0; i < currentLines.length; i++) {
            String lineText = currentLines[i];
            
            // Find stored color for this line, or use default
            ColorOption lineColor = defaultColor;
            if (i < messageLines.size()) {
                lineColor = messageLines.get(i).getColor();
            }
            
            // Create Text node with message content
            Text textNode = new Text(lineText);
            textNode.setFill(Paint.valueOf(lineColor.getColorCode()));
            textNode.setStyle("-fx-font-size: 14px; -fx-font-family: 'Courier New'; -fx-font-weight: bold;");
            
            messageDisplayFlow.getChildren().add(textNode);
            
            // Add line break if not the last line
            if (i < currentLines.length - 1) {
                Text newLine = new Text("\n");
                messageDisplayFlow.getChildren().add(newLine);
            }
        }
        
        // Scroll to bottom
        Platform.runLater(() -> {
            messageScrollPane.setVvalue(1.0);
        });
    }

    @FXML
    private void clearTextArea() {
        messageLines.clear();
        unifiedInputArea.clear();
        initializeMessageDisplay();
        resultLabel.setText("All messages cleared");
        resultLabel.setStyle("-fx-text-fill: #666666;");
        clearError();
        
        logger.info("All messages cleared by user");
    }

    private void loadButtonsFromService() {
        buttonPane.getChildren().clear();
        
        try {
            EJBServiceLocator serviceLocator = EJBServiceLocator.getInstance();
            DailyFileSLBeanRemote dailyFileSLBeanRemote = serviceLocator.retrieveDailyFileSLBeanEJB();
            
            List<String> buttonNames = dailyFileSLBeanRemote.getButtonNames();
            
            for (String buttonName : buttonNames) {
                Button button = new Button(buttonName);
                button.setOnAction(this::handleOptionSelected);
                button.setStyle("-fx-pref-width: 140; -fx-pref-height: 35; -fx-font-size: 11px; -fx-background-radius: 3;");
                buttonPane.getChildren().add(button);
            }
            
            logger.info("Loaded " + buttonNames.size() + " buttons from service");
            
        } catch (NamingException ex) {
            logger.severe("Failed to get EJB connection");
            logger.log(Level.SEVERE, ex.getMessage(), ex);
            showError("Failed to load predefined messages from database");
        } catch (Exception ex) {
            logger.log(Level.SEVERE, ex.getMessage(), ex);
            showError("Error loading predefined messages");
        }
    }

    private void showError(String message) {
        if (Platform.isFxApplicationThread()) {
            displayError(message);
        } else {
            Platform.runLater(() -> displayError(message));
        }
    }

    private void displayError(String message) {
        if (errorLabel != null) {
            errorLabel.setText("⚠ " + message);
            errorLabel.setVisible(true);
            errorLabel.setManaged(true);
            errorLabel.setStyle("-fx-text-fill: #dc3545; -fx-font-weight: bold; -fx-font-size: 12px;");

            PauseTransition pause = new PauseTransition(Duration.seconds(8));
            pause.setOnFinished(e -> {
                errorLabel.setVisible(false);
                errorLabel.setManaged(false);
                errorLabel.setText("");
            });
            pause.play();
        } else {
            if (resultLabel != null) {
                resultLabel.setText("⚠ Error: " + message);
                resultLabel.setStyle("-fx-text-fill: #dc3545; -fx-font-weight: bold;");
                
                PauseTransition pause = new PauseTransition(Duration.seconds(8));
                pause.setOnFinished(e -> {
                    resultLabel.setText("Ready");
                    resultLabel.setStyle("-fx-text-fill: #666666;");
                });
                pause.play();
            }
        }
        
        logger.warning("UI Error displayed: " + message);
    }

    private void clearError() {
        if (Platform.isFxApplicationThread()) {
            clearErrorDisplay();
        } else {
            Platform.runLater(() -> clearErrorDisplay());
        }
    }

    private void clearErrorDisplay() {
        if (errorLabel != null) {
            errorLabel.setVisible(false);
            errorLabel.setManaged(false);
            errorLabel.setText("");
        }
    }

    // Method to get all messages with their colors for backend processing
    public List<MessageData> getMessagesWithColors() {
        List<MessageData> messages = new ArrayList<>();
        for (MessageLine line : messageLines) {
            messages.add(new MessageData(line.getText(), line.getColor().getId()));
        }
        return messages;
    }

    // Method to get current message count
    public int getCurrentMessageCount() {
        return messageLines.size();
    }

    // Method to get formatted display text (for logging/debugging)
    public String getFormattedDisplayText() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < messageLines.size(); i++) {
            MessageLine line = messageLines.get(i);
            sb.append("[").append(line.getColor().getName()).append("] ").append(line.getText());
            if (i < messageLines.size() - 1) {
                sb.append("\n");
            }
        }
        return sb.toString();
    }

    // Inner class to store each line with its color and position
    private static class MessageLine {
        private String text;
        private ColorOption color;
        private int lineIndex;

        public MessageLine(String text, ColorOption color, int lineIndex) {
            this.text = text;
            this.color = color;
            this.lineIndex = lineIndex;
        }

        public String getText() { return text; }
        public ColorOption getColor() { return color; }
        public int getLineIndex() { return lineIndex; }
        
        @Override
        public String toString() {
            return "[" + color.getName() + "][Line " + lineIndex + "] " + text;
        }
    }

    // Color option class
    public static class ColorOption {
        private String name;
        private int id;
        private String colorCode;

        public ColorOption(String name, int id, String colorCode) {
            this.name = name;
            this.id = id;
            this.colorCode = colorCode;
        }

        public String getName() { return name; }
        public int getId() { return id; }
        public String getColorCode() { return colorCode; }

        @Override
        public String toString() {
            return name;
        }
    }

    // Data class for backend communication
    public static class MessageData {
        private String message;
        private int colorId;

        public MessageData(String message, int colorId) {
            this.message = message;
            this.colorId = colorId;
        }

        public String getMessage() { return message; }
        public int getColorId() { return colorId; }
        
        @Override
        public String toString() {
            return "MessageData{message='" + message + "', colorId=" + colorId + "}";
        }
    }
}
