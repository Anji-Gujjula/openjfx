@Override
public Integer postDisplayMessage(String buttonText, String loginId) {

    LOGGER.log(Level.INFO, "ENTER postDisplayMessage with buttonText: {0}, loginId: {1}", 
               new Object[]{buttonText, loginId});

    Integer boardId = null;
    Integer messageDetailId = null;
    LinkedHashMap<Integer, Object> parametersSet;
    Integer position;
    Timestamp userLogLoginTime = null;
    Timestamp boardLoginTime = null;

    // SQL to get login time from USER_LOG_TBL
    StringBuilder sqlGetUserLogTime = new StringBuilder();
    sqlGetUserLogTime.append(" SELECT LOGIN_TIME ");
    sqlGetUserLogTime.append(" FROM AVS.AVS_USER_LOG_TBL ");
    sqlGetUserLogTime.append(" WHERE LOGIN_ID = ? ");
    sqlGetUserLogTime.append(" ORDER BY LOGIN_TIME DESC ");
    sqlGetUserLogTime.append(" FETCH FIRST 1 ROWS ONLY ");

    // FIXED: SQL to check if user already exists in DISPLAY_BOARD_TBL and get LATEST login time
    StringBuilder sqlCheckUser = new StringBuilder();
    sqlCheckUser.append(" SELECT BOARD_ID, LOGIN_TIME ");
    sqlCheckUser.append(" FROM AVS.DISPLAY_BOARD_TBL ");
    sqlCheckUser.append(" WHERE LOGIN_ID = ? ");
    sqlCheckUser.append(" ORDER BY LOGIN_TIME DESC, BOARD_ID DESC ");
    sqlCheckUser.append(" FETCH FIRST 1 ROWS ONLY ");

    // SQL to insert new user login entry in board table
    StringBuilder sqlInsertBoard = new StringBuilder();
    sqlInsertBoard.append(" INSERT INTO AVS.DISPLAY_BOARD_TBL (BOARD_ID, LOGIN_ID, LOGIN_TIME) ");
    sqlInsertBoard.append(" VALUES (AVS.DISPLAY_BOARD_SEQ.NEXTVAL, ?, ?) ");

    // SQL to insert new entry when login time differs (keeping history)
    StringBuilder sqlInsertNewBoardEntry = new StringBuilder();
    sqlInsertNewBoardEntry.append(" INSERT INTO AVS.DISPLAY_BOARD_TBL (BOARD_ID, LOGIN_ID, LOGIN_TIME) ");
    sqlInsertNewBoardEntry.append(" VALUES (AVS.DISPLAY_BOARD_SEQ.NEXTVAL, ?, ?) ");

    // SQL to get the latest board_id for a user (most recent entry)
    StringBuilder sqlGetLatestBoardId = new StringBuilder();
    sqlGetLatestBoardId.append(" SELECT BOARD_ID ");
    sqlGetLatestBoardId.append(" FROM AVS.DISPLAY_BOARD_TBL ");
    sqlGetLatestBoardId.append(" WHERE LOGIN_ID = ? ");
    sqlGetLatestBoardId.append(" ORDER BY LOGIN_TIME DESC, BOARD_ID DESC ");
    sqlGetLatestBoardId.append(" FETCH FIRST 1 ROWS ONLY ");

    // SQL to insert message in child table
    StringBuilder sqlInsertMessage = new StringBuilder();
    sqlInsertMessage.append(" INSERT INTO AVS.DISPLAY_BOARD_MSG_DETAIL_TBL ");
    sqlInsertMessage.append(" (MESSAGE_DETAIL_ID, BOARD_ID, DISPLAY_MESSAGE, DISPLAY_MESSAGE_DATE_TIME) ");
    sqlInsertMessage.append(" VALUES (AVS.DISPLAY_BOARD_MSG_SEQ.NEXTVAL, ?, ?, SYSDATE) ");

    try (Connection conn = avsDatasource.getConnection();
         PreparedStatement pStmtGetUserLogTime = conn.prepareStatement(sqlGetUserLogTime.toString());
         PreparedStatement pStmtCheckUser = conn.prepareStatement(sqlCheckUser.toString());
         PreparedStatement pStmtInsertBoard = conn.prepareStatement(sqlInsertBoard.toString());
         PreparedStatement pStmtInsertNewBoardEntry = conn.prepareStatement(sqlInsertNewBoardEntry.toString());
         PreparedStatement pStmtGetLatestBoardId = conn.prepareStatement(sqlGetLatestBoardId.toString());
         PreparedStatement pStmtInsertMessage = conn.prepareStatement(sqlInsertMessage.toString())) {

        // Step 1: Get login time from AVS_USER_LOG_TBL
        LOGGER.log(Level.INFO, "Getting login time from USER_LOG_TBL - sql: {0}", 
                   sqlGetUserLogTime.toString().replaceAll("\\s+", " "));

        parametersSet = new LinkedHashMap<>();
        position = 1;
        parametersSet.put(position++, loginId);
        DBHelper.setParametersFromMap(pStmtGetUserLogTime, parametersSet);

        try (ResultSet resultSet = pStmtGetUserLogTime.executeQuery()) {
            if (resultSet.next()) {
                userLogLoginTime = resultSet.getTimestamp("LOGIN_TIME");
                LOGGER.log(Level.INFO, "Found login time in USER_LOG_TBL: {0}", userLogLoginTime);
            } else {
                LOGGER.log(Level.WARNING, "No login time found in USER_LOG_TBL for loginId: {0}", loginId);
                return null; // Cannot proceed without login time
            }
        }

        // Step 2: Check if user already exists in DISPLAY_BOARD_TBL (get LATEST entry)
        LOGGER.log(Level.INFO, "Checking if user exists in DISPLAY_BOARD_TBL (latest entry) - sql: {0}", 
                   sqlCheckUser.toString().replaceAll("\\s+", " "));

        parametersSet = new LinkedHashMap<>();
        position = 1;
        parametersSet.put(position++, loginId);
        DBHelper.setParametersFromMap(pStmtCheckUser, parametersSet);

        boolean userExists = false;
        try (ResultSet resultSet = pStmtCheckUser.executeQuery()) {
            if (resultSet.next()) {
                // User exists, get the LATEST board_id and login time
                boardId = resultSet.getInt("BOARD_ID");
                boardLoginTime = resultSet.getTimestamp("LOGIN_TIME");
                userExists = true;
                LOGGER.log(Level.INFO, "User exists with LATEST BOARD_ID: {0}, Latest Board Login Time: {1}", 
                          new Object[]{boardId, boardLoginTime});
            } else {
                LOGGER.log(Level.INFO, "No existing entries found for user: {0}", loginId);
            }
        }

        // Step 3: Handle user creation or update based on login time comparison
        if (!userExists) {
            // User doesn't exist, create new entry with login time from USER_LOG_TBL
            LOGGER.log(Level.INFO, "User doesn't exist, creating new entry - sql: {0}", 
                       sqlInsertBoard.toString().replaceAll("\\s+", " "));

            parametersSet = new LinkedHashMap<>();
            position = 1;
            parametersSet.put(position++, loginId);
            parametersSet.put(position++, userLogLoginTime);
            DBHelper.setParametersFromMap(pStmtInsertBoard, parametersSet);

            int insertCount = pStmtInsertBoard.executeUpdate();

            if (insertCount > 0) {
                LOGGER.log(Level.INFO, "New user entry created successfully");

                // Get the newly created board_id
                parametersSet = new LinkedHashMap<>();
                position = 1;
                parametersSet.put(position++, loginId);
                DBHelper.setParametersFromMap(pStmtGetLatestBoardId, parametersSet);

                try (ResultSet resultSet = pStmtGetLatestBoardId.executeQuery()) {
                    if (resultSet.next()) {
                        boardId = resultSet.getInt("BOARD_ID");
                        LOGGER.log(Level.INFO, "New user created with BOARD_ID: {0}", boardId);
                    }
                }
            } else {
                LOGGER.log(Level.WARNING, "Failed to create new user entry for loginId: {0}", loginId);
            }
        } else {
            // User exists, compare login times
            if (boardLoginTime != null && userLogLoginTime != null) {
                LOGGER.log(Level.INFO, "Comparing login times - Board: {0}, UserLog: {1}", 
                          new Object[]{boardLoginTime, userLogLoginTime});
                
                if (!boardLoginTime.equals(userLogLoginTime)) {
                    // Login times don't match, insert new entry with new board_id
                    LOGGER.log(Level.INFO, "Login times don't match. Board: {0}, UserLog: {1}. Creating new board entry - sql: {2}", 
                              new Object[]{boardLoginTime, userLogLoginTime, sqlInsertNewBoardEntry.toString().replaceAll("\\s+", " ")});

                    parametersSet = new LinkedHashMap<>();
                    position = 1;
                    parametersSet.put(position++, loginId);
                    parametersSet.put(position++, userLogLoginTime);
                    DBHelper.setParametersFromMap(pStmtInsertNewBoardEntry, parametersSet);

                    int insertCount = pStmtInsertNewBoardEntry.executeUpdate();

                    if (insertCount > 0) {
                        LOGGER.log(Level.INFO, "New board entry created successfully for different login time");

                        // Get the latest board_id (newly created one)
                        parametersSet = new LinkedHashMap<>();
                        position = 1;
                        parametersSet.put(position++, loginId);
                        DBHelper.setParametersFromMap(pStmtGetLatestBoardId, parametersSet);

                        try (ResultSet resultSet = pStmtGetLatestBoardId.executeQuery()) {
                            if (resultSet.next()) {
                                boardId = resultSet.getInt("BOARD_ID");
                                LOGGER.log(Level.INFO, "Using latest BOARD_ID for child tables: {0}", boardId);
                            }
                        }
                    } else {
                        LOGGER.log(Level.WARNING, "Failed to create new board entry for loginId: {0}", loginId);
                    }
                } else {
                    // Login times match, no new entry needed, use existing board_id
                    LOGGER.log(Level.INFO, "Login times match. Using existing BOARD_ID: {0} (no new board entry needed)", boardId);
                }
            } else {
                LOGGER.log(Level.WARNING, "Null login time detected. Board: {0}, UserLog: {1}", 
                          new Object[]{boardLoginTime, userLogLoginTime});
                // Handle null case - you might want to create a new entry or use existing boardId
                if (boardLoginTime == null && userLogLoginTime != null) {
                    LOGGER.log(Level.INFO, "Board login time is null but user log time exists. Creating new entry.");
                    // Create new entry logic here if needed
                }
            }
        }

        // Step 4: Insert button message in child table
        if (boardId != null) {
            LOGGER.log(Level.INFO, "Inserting button message for BOARD_ID: {0} - sql: {1}", 
                      new Object[]{boardId, sqlInsertMessage.toString().replaceAll("\\s+", " ")});
            
            parametersSet = new LinkedHashMap<>();
            position = 1;
            parametersSet.put(position++, boardId);
            parametersSet.put(position++, buttonText);
            DBHelper.setParametersFromMap(pStmtInsertMessage, parametersSet);
            
            int messageInsertCount = pStmtInsertMessage.executeUpdate();
            
            if (messageInsertCount > 0) {
                LOGGER.log(Level.INFO, "Button message inserted successfully for BOARD_ID: {0}, Message: {1}", 
                          new Object[]{boardId, buttonText});
                
                // Optional: Get the message_detail_id if needed
                String sqlGetMessageId = "SELECT MESSAGE_DETAIL_ID FROM AVS.DISPLAY_BOARD_MSG_DETAIL_TBL " +
                                       "WHERE BOARD_ID = ? AND DISPLAY_MESSAGE = ? " +
                                       "ORDER BY DISPLAY_MESSAGE_DATE_TIME DESC FETCH FIRST 1 ROWS ONLY";
                
                try (PreparedStatement pStmtGetMessageId = conn.prepareStatement(sqlGetMessageId)) {
                    parametersSet = new LinkedHashMap<>();
                    position = 1;
                    parametersSet.put(position++, boardId);
                    parametersSet.put(position++, buttonText);
                    DBHelper.setParametersFromMap(pStmtGetMessageId, parametersSet);
                    
                    try (ResultSet rs = pStmtGetMessageId.executeQuery()) {
                        if (rs.next()) {
                            messageDetailId = rs.getInt("MESSAGE_DETAIL_ID");
                            LOGGER.log(Level.INFO, "Message stored with MESSAGE_DETAIL_ID: {0}", messageDetailId);
                        }
                    }
                }
            } else {
                LOGGER.log(Level.WARNING, "Failed to insert button message for BOARD_ID: {0}", boardId);
            }
        } else {
            LOGGER.log(Level.SEVERE, "Failed to get or create BOARD_ID for loginId: {0}. Cannot insert message.", loginId);
        }
        
    } catch (SQLException ex) {
        LOGGER.log(Level.SEVERE, "SQL Exception occurred in postDisplayMessage: {0}", ex.getMessage());
        LOGGER.log(Level.SEVERE, ex.getMessage(), ex);
        ExceptionUtils.printRootCauseStackTrace(ex);
    } catch (Exception ex) {
        LOGGER.log(Level.SEVERE, "General Exception occurred in postDisplayMessage: {0}", ex.getMessage());
        LOGGER.log(Level.SEVERE, ex.getMessage(), ex);
        ExceptionUtils.printRootCauseStackTrace(ex);
    }
    
    LOGGER.log(Level.INFO, "EXIT postDisplayMessage with result - BOARD_ID: {0}, MESSAGE_DETAIL_ID: {1}", 
               new Object[]{boardId, messageDetailId});
    
    return boardId; // or return messageDetailId based on your needs
}
