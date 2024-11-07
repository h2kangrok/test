# test


import time
import serial
import requests
import numpy
from io import BytesIO
from pprint import pprint
from requests.auth import HTTPBasicAuth
from collections import Counter

import cv2
#########################################################################################################################################################################
colors = {'BOOTSEL' : (255, 0, 0), 'USB' : (255, 125, 0), 'CHIPSET' : (255, 255, 0), 'OSILLATOR' : (125, 255, 0), 'RASPBERRY PICO' : (0, 255, 0), 'HOLE' : (0, 0, 255)}
#########################################################################################################################################################################

ser = serial.Serial("/dev/ttyACM0", 9600)

# API endpoint
api_url = ""
#########################################################################################################################################################################

def make_textbox(image, text, startx, starty, color, font_color):
    text_size, _ = cv2.getTextSize(text, cv2.FONT_HERSHEY_SIMPLEX, 0.5, 1)  # Get text size
    text_width, text_height = text_size
    padding = 5  # Add some padding around the text

    # Coordinates for the rectangle
    x1_rect = startx
    y1_rect = starty - text_height - padding
    x2_rect = startx + text_width + padding
    y2_rect = starty + padding

    # Draw a filled rectangle behind the text
    cv2.rectangle(image, (x1_rect, y1_rect), (x2_rect, y2_rect), color, -1)
    cv2.putText(image, text, (startx, starty), cv2.FONT_HERSHEY_SIMPLEX, 0.5, font_color, 1, cv2.LINE_AA)
#########################################################################################################################################################################

def get_img():
    """Get Image From USB Camera

    Returns:
        numpy.array: Image numpy array
    """

    cam = cv2.VideoCapture(0)

    if not cam.isOpened():
        print("Camera Error")
        exit(-1)

    ret, img = cam.read()
    cam.release()

    return img


def crop_img(img, size_dict):
    x = size_dict["x"]
    y = size_dict["y"]
    w = size_dict["width"]
    h = size_dict["height"]
    img = img[y : y + h, x : x + w]
    return img


def inference_reqeust(img: numpy.array, api_rul: str):
    """_summary_

    Args:
        img (numpy.array): Image numpy array
        api_rul (str): API URL. Inference Endpoint
    """
    _, img_encoded = cv2.imencode(".jpg", img)

    # Prepare the image for sending
    img_bytes = BytesIO(img_encoded.tobytes())

    # Send the image to the API
    files = {"file": ("image.jpg", img_bytes, "image/jpeg")}

    print(files)

    try:
        response = requests.post(api_url, files=files)
        if response.status_code == 200:
            pprint(response.json())
            return response.json()
            print("Image sent successfully")
        else:
            print(f"Failed to send image. Status code: {response.status_code}")
    except requests.exceptions.RequestException as e:
        print(f"Error sending request: {e}")


while 1:
    data = ser.read()
    print(data)
    if data == b"0":
        img = get_img()
        # crop_info = None
        crop_info = {"x": 280, "y": 180, "width": 230, "height": 260}

        if crop_info is not None:
            img = crop_img(img, crop_info)

#########################################################################################################################################################################

        response = requests.post(
            url="https://suite-endpoint-api-apne2.superb-ai.com/endpoints/feed26fc-ce91-451d-89f3-8de7f4c880d9/inference",
            auth=HTTPBasicAuth("kdt2024_1-25", "tgQ2dmUkRR83hlNogvILG2eiD0Cfay6G52Hb8P7R"),
            headers={"Content-Type": "image/jpeg"},
            data=img,
        )
        aaa = response.json()

        print(aaa)

        for obj in aaa['objects']:
            box = obj['box']
            start_point = (box[0], box[1])
            end_point = (box[2], box[3])
            color = colors[obj['class']]
            thickness = 2
            cv2.rectangle(img, start_point, end_point, color, thickness)

            text = obj['class'] + ' score :' + str(round(obj['score'], 2))

            make_textbox(img, text, box[0], box[1], color, (255, 255, 255))

        count_dict = dict(Counter(d["class"] for d in aaa['objects']))

        y1 = 20
        x1 = 0   
        for key in count_dict.keys():
            text = key + ":" + str(count_dict[key])

            make_textbox(img, text, x1, y1, (0, 255, 0), (0, 0, 0))
            y1 += 20
        
        text =  "total :" + str(sum(count_dict.values()))

        make_textbox(img, text, x1, y1, (0, 255, 0), (0, 0, 0))
        
#########################################################################################################################################################################

        cv2.imshow("", img)
        cv2.waitKey(1)
        result = inference_reqeust(img, api_url)
        ser.write(b"1")
    else:
        pass
