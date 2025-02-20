% Load video
videoFile = 'FILEPATH';
vidObj = VideoReader(videoFile);

% Get frame rate
frameRate = vidObj.FrameRate;
dt = 1 / frameRate; % Time difference between frames

% Define output folder for saving arrays
outputFolder = '**************************************';

% Ensure the folder exists
if ~exist(outputFolder, 'dir')
    mkdir(outputFolder);
end

% Initialize arrays to store positions
redPositions = [];
greenPositions = [];
timestamps = [];

frameIdx = 0; % Frame counter

% Process each frame in the video
while hasFrame(vidObj)
    frame = readFrame(vidObj);
    frameIdx = frameIdx + 1; % Update frame index
    
    % Store timestamp for reference
    timestamps = [timestamps; (frameIdx - 1) * dt];

    % Extract red and green channels
    redChannel = frame(:,:,1);
    greenChannel = frame(:,:,2);

    % Apply thresholding
    levelRed = 150;
    levelGreen = 150;

    redMask = redChannel > levelRed;
    greenMask = greenChannel > levelGreen;

    % Find centroids of detected LEDs
    redStats = regionprops(redMask, 'Centroid');
    greenStats = regionprops(greenMask, 'Centroid');

    % Store red LED position
    if ~isempty(redStats)
        redPositions = [redPositions; redStats(1).Centroid];
    else
        redPositions = [redPositions; NaN, NaN]; % If red LED is not found
    end

    % Store green LED position
    if ~isempty(greenStats)
        greenPositions = [greenPositions; greenStats(1).Centroid];
    else
        greenPositions = [greenPositions; NaN, NaN]; % If green LED is not found
    end
end

% Save position data
save(fullfile(outputFolder, 'Red_Positions.mat'), 'redPositions');
save(fullfile(outputFolder, 'Green_Positions.mat'), 'greenPositions');
save(fullfile(outputFolder, 'Timestamps.mat'), 'timestamps');

writematrix(redPositions, fullfile(outputFolder, 'Red_Positions.csv'));
writematrix(greenPositions, fullfile(outputFolder, 'Green_Positions.csv'));
writematrix(timestamps, fullfile(outputFolder, 'Timestamps.csv'));

disp('Processing complete! Positions and timestamps saved.');

%% DISPLACEMENT AND VELOCITY CALCULATION

% Ensure valid data (both LEDs detected)
validFrames = ~isnan(redPositions(:,1)) & ~isnan(greenPositions(:,1));

% Compute displacement (Red-Green distance over time)
displacement = sqrt((redPositions(validFrames,1) - greenPositions(validFrames,1)).^2 + ...
                    (redPositions(validFrames,2) - greenPositions(validFrames,2)).^2);

% Compute velocity (change in displacement over time)
velocity = [0; diff(displacement)] / dt;  % Velocity in pixels/sec

% Save results
save(fullfile(outputFolder, 'Displacement.mat'), 'displacement');
save(fullfile(outputFolder, 'Velocity.mat'), 'velocity');

writematrix(displacement, fullfile(outputFolder, 'Displacement.csv'));
writematrix(velocity, fullfile(outputFolder, 'Velocity.csv'));

disp('Displacement and velocity calculated and saved.');

%% PLOT DISPLACEMENT AND VELOCITY OVER TIME
figure;

% Displacement Plot
subplot(2,1,1);
plot(timestamps(validFrames), displacement, 'b-', 'LineWidth', 2);
xlabel('Time (seconds)');
ylabel('Displacement (pixels)');
title('Red-Green LED Displacement Over Time');
grid on;

% Velocity Plot
subplot(2,1,2);
plot(timestamps(validFrames(2:end)), velocity, 'm-', 'LineWidth', 2);
xlabel('Time (seconds)');
ylabel('Velocity (pixels/sec)');
title('Velocity Over Time');
grid on;
