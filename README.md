<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.*?>
<?import javafx.scene.layout.*?>
<?import javafx.scene.control.*?>
<?import javafx.geometry.*?>
<?import javafx.scene.paint.*?>
<?import javafx.scene.text.*?>
<?import javafx.scene.control.cell.PropertyValueFactory?>
<?import java.time.format.DateTimeFormatter?>
<?import javafx.scene.control.Button?>
<?import javafx.scene.control.Label?>
<?import javafx.scene.layout.VBox?>
<?import javafx.scene.layout.FlowPane?>
<?import javafx.geometry.Insets?>
<?import gov.ca.lc.controllers.cellfactories.DailyFileNewSessionDateFormatConverter?>

<VBox xmlns:fx="http://javafx.com/fxml/1" fx:controller="gov.ca.lc.controllers.admin.DisplayMessageController" spacing="0" alignment="TOP_LEFT" style="-fx-padding: 0;">
    <VBox fx:id="dailyFilePanelVBox" alignment="TOP_CENTER" spacing="10" VBox.vgrow="ALWAYS" style="-fx-background-color:#FFFFFF;">
        <padding>
            <Insets top="12.0" left="25.0" right="25.0" bottom="12.0"/>
        </padding>
        
        <HBox fx:id="newSessionHBox" fillHeight="true" HBox.hgrow="ALWAYS" spacing="10" alignment="CENTER_LEFT">
            <HBox.margin>
                <javafx.geometry.Insets>
                    <left>15</left>
                </javafx.geometry.Insets>
            </HBox.margin>
        </HBox>
        
        <!-- Color Selection -->
        <HBox spacing="10" alignment="CENTER_LEFT">
            <Label text="Select Color:" style="-fx-font-weight: bold;"/>
            <ComboBox fx:id="colorComboBox" prefWidth="150"/>
        </HBox>
        
        <Label text="Select an option:" style="-fx-font-weight: bold;"/>
        
        <!-- FlowPane for your existing buttons (from DB) -->
        <FlowPane fx:id="buttonPane" hgap="10" vgap="10">
            <padding>
                <Insets top="5" right="0" bottom="5" left="0"/>
            </padding>
        </FlowPane>
        
        <Label fx:id="resultLabel" text="Nothing selected yet"/>
        
        <!-- Message Display and Input Area -->
        <VBox spacing="8" VBox.vgrow="ALWAYS">
            <!-- Color Preview Area (TextFlow) -->
            <VBox spacing="5">
                <Label text="Message Preview (with colors):" style="-fx-font-weight: bold; -fx-font-size: 12px; -fx-text-fill: #666666;"/>
                <ScrollPane fx:id="messageScrollPane" 
                           prefHeight="120" 
                           fitToWidth="true"
                           style="-fx-background-color: #f8f9fa; -fx-background-radius: 3; -fx-border-color: #dee2e6; -fx-border-radius: 3; -fx-border-width: 1;">
                    <TextFlow fx:id="messageDisplayFlow" 
                             style="-fx-background-color: #ffffff; -fx-padding: 8; -fx-line-spacing: 2px;"/>
                </ScrollPane>
            </VBox>
            
            <!-- Input/Edit Area -->
            <VBox spacing="5">
                <Label text="Create Custom Message:" style="-fx-font-weight: bold; -fx-font-size: 14px;"/>
                <TextArea fx:id="unifiedInputArea" 
                         prefHeight="200" 
                         promptText="Type your message here or click buttons above..."
                         style="-fx-font-size: 14px;" 
                         wrapText="true"/>
            </VBox>
        </VBox>
        
        <!-- Error Display -->
        <Label fx:id="errorLabel" 
               text="" 
               style="-fx-text-fill: red; -fx-font-weight: bold; -fx-font-size: 12px;"
               visible="false" 
               managed="false"/>
        
        <!-- Action Buttons -->
        <HBox spacing="10" alignment="CENTER_RIGHT">
            <Button fx:id="clearButton" 
                    text="Clear All" 
                    onAction="#clearTextArea" 
                    prefWidth="100" 
                    prefHeight="40"/>
            <Button fx:id="sendButton" 
                    text="Apply Colors" 
                    onAction="#processCustomText"
                    prefWidth="120" 
                    prefHeight="40"/>
        </HBox> 
        
    </VBox>
</VBox>
