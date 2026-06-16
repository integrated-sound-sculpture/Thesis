# Source Code for Visualisation
```python
#import pyqtgraph.examples
import ctypes
#ctypes.windll.ole32.OleInitialize(None)

from scipy.io import wavfile
import numpy as np
import pyqtgraph as pg
import sys
# import sounddevice as sd
import threading
from scipy.fft import fft, fftfreq
from scipy.signal import spectrogram
# import soundfile as sf
import socket
from pyqtgraph.Qt import QtCore, QtWidgets, QtGui
import time as tm


sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)                         #UART connection
sock.bind(("127.0.0.1", 8080))
sock.setblocking(False)

sample_rate = 48000

app = QtWidgets.QApplication(sys.argv)                                          #start the application                       

chunk_size = 2048                                           
time = np.arange(chunk_size) / sample_rate                                      #time array for one chunk, used for plotting the waveform
chunk = np.zeros(chunk_size)                                                    #audio data chunk
current_chunk = chunk.copy()                                                    #copy of the current chunk, used as a register

hann_window = np.hanning(chunk_size)                                            #hann window for the spectrogram, to reduce spectral leakage
windowed_chunk = chunk * hann_window                                            #apply the window to the chunk before performing the FFT

fft_values = np.abs(np.array(fft(windowed_chunk)))[:chunk_size//2]              #FFT values for the chunk // 2 to only take positive frequencies
fft_freqs = fftfreq(chunk_size, 1/sample_rate)[:chunk_size//2]                  #frequency corresponding to fft_values

win = QtWidgets.QWidget()                                                       #window setup
win.setWindowTitle('Audio Visualisation')
layout = QtWidgets.QVBoxLayout()
win.setLayout(layout)

selector = QtWidgets.QComboBox()                                                #create dropdown selector
selector.addItems(['All', 'Waveform plot', 'FFT', 'Frequency plot','Piano'])
layout.addWidget(selector)
last_selector = 0

p1_ymax = 0.1                                                                   #initial y-axis max's, will be adjucted in update()
p2_ymax = 0.1 
p3_fft_ymax = 0.1  

f_test, t_test, Sxx_test = spectrogram(chunk, fs=sample_rate, nperseg=512, noverlap=256)   #only f-test is used, nperseg is window size, noverlap is amount op points that overlap between windows
n_freqs = len(f_test)                                                           # 512 / 2 + 1 = 257, single sided, including the zero
waterfall_buffer = np.zeros((n_freqs, 200)) - 80                                 # Making a bbuffer for the waterfall spectrogram, -80 to make the first few frames dark
log_freqs = np.logspace(np.log10(20), np.log10(sample_rate//2), n_freqs)        #log frequencies from 20 untill nyquist frequency

freq_labels = [20, 50, 100, 200, 500, 1000, 2000, 5000, 10000, 20000]           # frequencie labels.
tick_positions = [(np.log10(f), f'{f} Hz') for f in freq_labels]                #afstand van fft_freqs tot de frequenties
tick_positions_log = [(int(np.argmin(np.abs(log_freqs - f))), f'{f}') for f in freq_labels]  #afstand van fft_freqs tot de frequenties

note_labels = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'] #Note labels, for some reason it starts with C
midi_min = 21                                                                   #Minimal MIDI value of a piano
midi_max = 108                                                                  #Maximal MIDI value of a piano
midi_freqs = 440 * 2 ** ((np.arange(midi_min, midi_max + 1) - 69) / 12)         #freq die bij elke midi toon hoort

amplitude_threshold = 5                                                         #threshold to suppress noise

p1 = pg.PlotWidget(title= 'Waveform plot')                                               #plot waveform
p1.setLabel('bottom', 'Time', units='s')
p1.setLabel('left', 'Amplitude')
graph1 = p1.plot(time.flatten(), chunk.astype(float).flatten())                 #flatten to use with pyqtgraph
p1.setXRange(0,time[-1])
p1.enableAutoRange(axis='x', enable=False)
layout.addWidget(p1)

p2 = pg.PlotWidget(title= 'FFT')                                                #plot FFT     
p2.setLabel('bottom', 'Frequency', units='Hz')
p2.setLabel('left', 'Magnitude')
graph2 = p2.plot(fft_freqs.flatten(), fft_values.flatten())
p2.setXRange(20, sample_rate//2)                                                #limit x-axis to Nyquist frequency
p2.setLogMode(x=True, y=False)                                                  #set logarithmic x-axis
tick_positions = [(np.log10(f), f'{f} Hz') for f in freq_labels]                #distance from fft_freqs to frequencies
p2.hide()
layout.addWidget(p2)

p3_packet = QtWidgets.QWidget()                                                 #create a packet for the plotting
p3_layout = QtWidgets.QVBoxLayout()                                             #layout voor plots onder elkaar
p3_packet.setLayout(p3_layout)                                                  #put the layout in the packet

p3_fft = pg.PlotWidget(title= 'Frequency plot')                                            #plot FFT of the packet
p3_fft.setLabel('left', 'Magnitude')
p3_fft.getAxis('left').setWidth(60)                                             #a constant width for y axis to ensure both y axis are on the same place
graph3_fft = p3_fft.plot(fft_freqs.flatten(), fft_values.flatten())             #flatten to use with pyqtgraph
p3_fft.getAxis('bottom').setStyle(showValues=False)                             #hide x-axis labels for the fft plot in the packet becasue they are shown in the spectrogram
p3_fft.setYRange(0, p3_fft_ymax)                                                #initial y-axis range
p3_layout.addWidget(p3_fft)

p3 = pg.PlotWidget()                                                            #plot spectrogram
p3.setLabel('left', 'Time (frames)')
p3.setLabel('bottom', 'Frequency', units='Hz')
p3.getAxis('left').setWidth(60)
graph3 = pg.ImageItem()
p3.addItem(graph3)
graph3.setColorMap('inferno')                                                   #colormap for the spectrogram, other option is viridis

p3.getAxis('bottom').setTicks([tick_positions_log])                             #set log frequency labels on the x-axis
p3_layout.addWidget(p3)
layout.addWidget(p3_packet)

p3_fft.setXLink(p3)                                                             #link the axis together
p3_packet.hide()

p4 = pg.PlotWidget(title= 'Notes')                                              #plot note
p4.setAspectLocked(False)
p4.setYRange(0, 1)
p4.setXRange(0, 88)
p4.getAxis('bottom').setStyle(showValues=False)
p4.getAxis('left').setStyle(showValues=False)
p4.hide()

white_keys = []
black_keys = []
white_key_items = {}
black_key_items = {}

white_pattern = [0,2,4,5,7,9,11]                                    # to connect midi notes to the right color key
black_pattern = [1,3,6,8,10]

white_x = 0
white_positions = {}
for midi in range(midi_min, midi_max + 1):                          #loop through midi notes and assign positions for white keys
    note = midi % 12
    if note in white_pattern:
        white_positions[midi] = white_x
        white_x += 1                            

for midi, x in white_positions.items():                             #create rectangles for white keys based
    rect = QtWidgets.QGraphicsRectItem(x, 0, 1, 1)
    rect.setBrush(pg.mkBrush('w')  )
    rect.setPen(pg.mkPen('k'))
    p4.addItem(rect)
    white_key_items[midi] = rect

for midi in range(midi_min, midi_max + 1):                          #loop through midi notes and create rectangles for black keys, offset calculated based on preious white key
    note = midi % 12
    if note in black_pattern:
        prev_white_x = midi - 1
        if prev_white_x in white_positions:
            x = white_positions[prev_white_x] + 0.6
            rect = QtWidgets.QGraphicsRectItem(x, 0.4, 0.6, 0.6)    #offset of black key
            rect.setBrush(pg.mkBrush('k')  )
            rect.setPen(pg.mkPen('k'))
            p4.addItem(rect)
            black_key_items[midi] = rect
p4.setXRange(0, white_x)
layout.addWidget(p4)

def on_select(index):                                               #function to switch between plots
    if index == 0:
        p1.show()
        p2.hide()
        p3_packet.show()
        p4.hide()
    elif index == 1:
        p1.show()
        p2.hide()
        p3_packet.hide()
        p4.hide()
    elif index == 2:
        p1.hide()
        p2.show()
        p3_packet.hide()
        p4.hide()
    elif index == 3:
        p1.hide()
        p2.hide()
        p3_packet.show()
        p4.hide()
    else:
        p1.hide()
        p2.hide()
        p3_packet.hide()
        p4.show()

selector.currentIndexChanged.connect(on_select)

smooth_midi = [69.0]

def update():                                                       #update function for the plots, called by a timer
    global waterfall_buffer, chunk, p1_ymax, p2_ymax, p3_fft_ymax, last_selector
    start = tm.perf_counter()

    data = None
    try:
        while True:
            data, addr = sock.recvfrom(chunk_size * 2)
    except BlockingIOError:
        pass

    if data is None:
        return


    view_selector = data[0]    
    chunk = np.frombuffer(data[1:], dtype=np.int16)

    if len(chunk) < chunk_size:
        return

    chunk = chunk / np.max(np.abs(chunk))

    windowed_chunk = chunk * hann_window
    if view_selector != last_selector:
        selector.setCurrentIndex({0:1,1:3,2:4}.get(view_selector, 0))  # Update the selector based on the received data
        last_selector = view_selector


    fft_values = np.abs(np.array(fft(windowed_chunk)))[:chunk_size//2]       #perform fft, put it in an array and take the absolute value, for only positive frequencies
    fft_freqs = fftfreq(chunk_size, 1/sample_rate)[:chunk_size//2]  #get corresponding frequencies for the fft values
    fft_values_log = np.interp(log_freqs, fft_freqs, fft_values)    #interpolate the fft values to match the log frequency bins

    f,t,Sxx = spectrogram(chunk, fs=sample_rate, nperseg =512, noverlap=256, window ='hann') #make spectrogram
    magnitude_db_spec = 10 * np.log10(Sxx + 1e-10)                  #convert magnitude to logaritmic, avoid log(0)
    col = magnitude_db_spec.mean(axis=1)                            #average over time to get a single column for the waterfall plot    
    col_log = np.interp(log_freqs, f, col)                          #interpolate to match the log frequency bins
    graph1.setData(time.flatten(), chunk.astype(float).flatten())   #update waveform plot
    graph2.setData(fft_freqs.flatten(), fft_values.flatten())       #update FFT plot
    graph3_fft.setData(np.arange(len(log_freqs)), fft_values_log)
    waterfall_buffer = np.roll(waterfall_buffer, -1, axis=1)        #shift buffer to the left
    waterfall_buffer[:, -1] = col_log                               #add new column to the right of the buffer  
    graph3.setImage(waterfall_buffer, autoLevels=True)

    if np.max(np.abs(chunk)) > p1_ymax:                             #dynamically adjust y-axis if the signal exceeds 90% of the current max
        p1_ymax = p1_ymax * 1.5
        p1.setYRange(-p1_ymax, p1_ymax)
    if np.max(fft_values) > p2_ymax:
        p2_ymax = p2_ymax * 1.5
        p2.setYRange(0, p2_ymax)
    if np.max(fft_values) > p3_fft_ymax:
        p3_fft_ymax = p3_fft_ymax * 1.5
        p3_fft.setYRange(0, p3_fft_ymax)

    valid_freq = fft_freqs > 20
    if np.any(valid_freq):
        valid_fft = fft_values[valid_freq]
        valid_ffreqs = fft_freqs[valid_freq]
        peak_idx = np.argmax(valid_fft)                             #find the index of the peak in the FFT
        peak_freq = fft_freqs[valid_freq][peak_idx]                 #get the corresponding frequency
        peak_magnitude = valid_fft[peak_idx]                        #get the magnitude of the peak
        
        if 0 < peak_idx < len(valid_fft) -1:
            left = valid_fft[peak_idx-1]
            middle = valid_fft[peak_idx]
            right = valid_fft[peak_idx+1]
            correction = 0.5*(left-right)/(left-2*middle+right)
            bin_width = fft_freqs[1] - fft_freqs[0]
            peak_freq = valid_ffreqs[peak_idx] + correction * bin_width
        else:
            peak_freq = valid_ffreqs[peak_idx]

        midi_note = 69 + 12 * np.log2(max(peak_freq / 440, 1e-10))  #convert frequency to MIDI note number

        center = round(midi_note)                                   #get the nearest MIDI note number to the smoothed value

        for midi, rect in white_key_items.items():                  #reset all colors
            rect.setBrush(pg.mkBrush('w'))
        for midi, rect in black_key_items.items():
            rect.setBrush(pg.mkBrush('k'))

        if peak_magnitude > amplitude_threshold:                    #only show the detected note if it exceeds the amplitude threshold
            if center in white_key_items:                           #highlight the note by making the key red
                white_key_items[center].setBrush(pg.mkBrush('r'))
            elif center in black_key_items:
                black_key_items[center].setBrush(pg.mkBrush('r'))
            note_name = note_labels[center % 12]                    #get the name of the detected note
            octave = center // 12 - 1
            p4.setTitle(f'Detected Note: {note_name}{octave}')      #set the title to show the detected note and its frequency and amplitude
    end = tm.perf_counter()
    processing_time = (end-start)*1000
    p1.setTitle(f'Delay: {processing_time:.2f}ms')
    QtCore.QTimer.singleShot(5, update)

        
QtCore.QTimer.singleShot(0, update)

win.showMaximized()                                                 #show the window fullscreen
win.show()
app.exec()
```