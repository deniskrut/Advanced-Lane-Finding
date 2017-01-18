# Steps to achieve the result

## Camera calibration

I’ve performed camera calibration on given set of images.

## Obtain a binary image

I’ve observed that detection works better on S channel of HLS on some set of images, while on others L works better. So I used OR combination to get both. For each channel I picked pixels where both x and y Sobel thresholds are present, or gradient magnitude with gradient direction threshold. I used “and” to make sure I can exclude false positives, while I used “or” to get more true positives.

## Perspective transform

I used perspective transform to get birds eye view of the road. Because road goes up and down, the view was not perfect every time. Road also can get blurry closer to the horizon. I picked a rectangle that gave me not too blurry road and worked decently on descending and ascending roads.

## Finding lanes

If I don’t have prior assumptions about where the lanes are, I do a full search. First I cut image on to 8 pieces horizontally. For each piece I get a histogram and peaks on that histogram. Form that peaks I form a pairs of peaks that are likely a lanes - based on distance between the peaks. I sort those pairs in a way that lanes containing highest peaks go first. Let’s call those pairs a candidates for lanes. I also add candidates for the cases when neither lanes are visible or only one lane is visible.

Later I start with some group of candidates, and perform a successive refinement of that group. I try to replace one element in that group with another element that will give me a better lanes in result. To define which lanes are better, I wrote a scoring function. That function favors lanes that are parallel, have about the same bend, slope, greater number of points and lower residual from the polynomial fit for the points obtained from that group.

The process above stops when I find good enough points. I have certain thresholds for what I believe good enough lanes are in terms of bend, slope, how close to a parallel they are and how much residual did I get from the polynomial fit. At that point I declare those points a lane and return. Otherwise, I return the best result I have been able to obtain.

## Warp lines back to original image

To warp lanes back to original image, I perform a drawing of lanes in the bird eye view image, and then transform it back to a normal view.

## Video pipeline

In order to get video to work, I have declared a global variables to pass results from previous frame to the next frame. If it is a first frame, or previous frame did not fit the criteria for good lanes - e.g. parallel, low residual, etc. - I start search from scratch as described above. Otherwise I perform a informed search.

To do informed search I take a polynomial fit from the previous image, cut given image horizontally on 8 pieces, and try to find lane points close to polynomial fit from the previous frame. Then I check the result makes a good lane and return, otherwise perform a full search on a given frame. 

Beauty of this method is in the fact that I will only be able to do successful informed search if I found actual lanes. If I found something else, it will not stay in about the same place in the next frame. So I get to try to do another full search on the next frame. There are exceptions though, such as vertical lines on the road.

Finally we calculate the curvature of the road and distance from center. That is performed using a polynomial fit for the lane points. To get a curvature I can get a derivative from the polynomial fit, given that fit was obtained in the real world coordinate system. To get a distance from center I can calculate bottom points of the polynomial fits of each lane, and measure the distance from center of the image to each of the points. By comparing the distance I can tell how far the car is from the center.

# Room for improvement

If I had more time, I would implement a better binary image processing. I think better results can be achieved by trying more color models and channels. This would give me better results on challenge videos, that can fail due to false positives in the binary image

I also could partially remove assumptions I gave for a good lanes to accommodate better for harder challenge.

Finally I can incorporate knowledge of the flow - e.g. which way we are moving. I should be able to tell which lanes roughly stay through out multiple frames, and which one are coming and going.

# How current pipeline can fail

Current pipeline fails if assumptions are violated - e.g. turn is so steep that only one lane is visible, while we try to find both lanes. Certain road colors can add false positives to the picture, causing pipeline to fail.