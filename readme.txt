The folder contains two files of 27 minutes data each:
    |- sensor-27minutes.csv     # the raw sensor data file with sampling rate 100Hz
    |- label-27minutes.csv      # the corresppinding label of four system parameters (S, D, H, R). 
                                # the timestamp of each row is the sum of the 2nd row's time and the corresponding 
                                # TimeStamp difference (in millisecond) between the current row and the 2nd row. 
                                # The example is calculated in column H.


Write a stream data analytics program to build the relationship model between the sensor data and the four parameters (S, D, H, R), 
and use the built model to predict the future four parameters per second from future raw sensor data. Python language 
is recommended for the stream data analytics program.

Please use first 20 minutes data for training, validation and testing. The remaining 7 minutes 
data is used as the new data to calculate/predict each of the four parameters per second and evaluate the MAE (mean absolute error) of 
each parameter per second. 

Grading: score = 100 - [MAE(S) + MAE(D) + MAE(H) + MAE(R)], where MAE(X) is the average of per-second MAE of parameter X during 
the final 7 minutes. For example, MAE(S) = average(abs(S_i-S'_i)), where S_i and S'_i are the predicted and labelled value of 
i-th second system parameter S.

Note: the stream data analytics program shall not be trivial or hardcoded (such as giving a fixed output). It must be a meaningful AI program
based on signal processing, or machine/deep learning, or statistics, etc.


Tips:
1. The stream data analytics program will be a combination of signal processing and machine/deep learning regresssion model;
2. The sensor data is seismocardiogram (SCG) data. Those four parameters are related to the features of peaks and envelopes 
    (see SCG.jpg for details), thereafter extracting them will help;
3. There are some abnorml events/interferences or missing data during some periods (which is normal in real-world sensor data), 
    the written program shall automatically identify and remove them; 
4. A potential approach:
4.1 Perform a low-pass filter and/or wavelet denoising first;
4.2 Extract stable envelopes and feature points, and perform regression analysis
4.2.1 First design a method that will yield stable envelopes by observing the envelopes in Grafana (not just in matplotlib); Sometime a method works for some data, but not stable, thus using Grafana to see behavior over a longer duration is necessary;
Potential method to generate stable envelopes first needs to remove abnorml events/interferences periods (through anomaly detection);
autocorrelation, similarity score and matrix profile may be used to search vital sign alike signals;
Matrix profile methods for time series: https://www.cs.ucr.edu/~eamonn/MatrixProfile.html 
Source code: https://stumpy.readthedocs.io/en/latest/Tutorial_Pattern_Searching.html 
4.2.2 Apply VMD, SWT or other decomposition methods to extract envolope;
Standalone heartbeat extraction in SCG signal using Variational Mode Decomposition: https://ieeexplore.ieee.org/document/8538723 
4.2.3 Extract feature points from autocorrelation; or stacking signals with a moving 1-minute window, then extracting feature points;
Automatic Annotation of peaks in Seismocardiogram for Systolic Time Intervals https://ieeexplore-ieee-org.proxy-remote.galib.uga.edu/stamp/stamp.jsp?tp=&arnumber=7591280
4.2.4 Start with a simple linear regression model that is based on the time difference between peaks and other feature points;

