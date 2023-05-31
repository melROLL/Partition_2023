# Partition_2023
a collection of Partition, Score, sheet music 

````
#!/bin/bash

# The script starts with shebang, which specifies the interpreter (Bash) to be used for executing the script.

# Melvyn ROLLAND 30/05/2023

# The script utilizes an API to upload photos from a folder to an online photo server. An API (Application Programming Interface) is protocols that makes software applications to communicate and interact with each other.
# update: it removes any photos from the server that are no longer present in the folder.


# The script checks if a specific folder is properly mounted.
# If it is, it connects to a server using an API to perform the following tasks:
#   - Upload photos from the folder to an online photo server.
#   - Remove any photos from the server that are no longer present in the folder.
# The script requires authentication credentials (user, password, and token) to access the API.
# The script also utilizes the 'jq' tool for JSON processing and 'exiftool' to extract metadata.
# Upon completion, the script logs out from the server and exits with an appropriate status code.
# If the folder is not properly mounted, the script exits with a different status code.

# Set the username, password, and token for authentication
user="mrolland" 
password="we4reDATA" #this is the goodpassword
altpassword="mel"
token="dlJXibPCipj5TzIcFu_6Zg=="

# Check if the folder is properly mounted
if mount | grep /home/datasource > /dev/null; then
    echo "The shared folder is properly mounted."

    # Connecting to the server.
    curl -X POST -H "Authorization: $token" \
    -H 'Content-Type: application/json' -H 'Accept: application/json' \
    -d "{\"username\": \"$user\", \"password\": \"$password\"}" \
    https://photoserver.mde.epf.fr/api/Session::login

    # Get the ID of the album associated with the token.
    id_album=$(curl -X POST -H "Authorization: $token" \
    -H 'Content-Type: application/json' -H 'Accept: application/json' \
    https://photoserver.mde.epf.fr/api/Albums::get | jq ".albums[] | .id")
    echo "The ID of your album is: $id_album"

    # Get the names of the photos in the album.
    photos_alb=$(curl -X POST -H "Authorization: $token" \
    -H 'Content-Type: application/json' -H 'Accept: application/json' \
    -d "{\"albumID\": $id_album}" \
    https://photoserver.mde.epf.fr/api/Album::get | jq '.photos[] | .title')

    # Clean the photo names.
    photos_alb_flt=$(echo $photos_alb | tr -d '"')

    echo "Processing in progress..."
    # For each file with the .jpg extension in the shared folder.
    for photo_fold in /home/datasource/*.jpg ; do
        # Clean the photo name.
        photo_fold_flt="$(echo ${photo_fold##*/} | cut -d'.' -f 1)"

        # The photo has not been found yet.
        photo_in_alb=0

        # If the photo in the folder is found in the album.
        if echo "$photos_alb_flt" | grep -qw "$photo_fold_flt" ; then
            # The photo has been found.
            photo_in_alb=1
        fi

        # If the photo has not been found.
        if ! ((photo_in_alb)) ; then
            # Extract the metadata.
            metadata=$(exiftool -j "/home/datasource/$photo_fold_flt.jpg")

            # Add the photo to the album and get its ID.
            id_photo_add=$(curl -X POST -H "Authorization: $token" \
            -H 'Content-Type: multipart/form-data' -H 'Accept: application/json' \
            -F "albumID=$id_album" \
            -F "file=@/home/datasource/$photo_fold_flt.jpg" \
            https://photoserver.mde.epf.fr/api/Photo::add | jq ".id")

            # Launch the program to get the associated color.
            color=$(/home/datasource/getColor.py \
            /home/datasource/$photo_fold_flt.jpg)
            # Replace the comma with the tag structure.
            color_flt=$(echo "$color" | sed 's/,/", "/g' | sed 's/^/"/;s/$/"/')

            # Inject the tags.
            curl -X POST -H "Authorization: $token" \
            -H 'Content-Type: application/json' -H 'Accept: application/json' \
            -d "{\"shall_override\": true, \"photoIDs\": [$id_photo_add], \"tags\": [$color_flt]}" \
            https://photoserver.mde.epf.fr/api/Photo::setTags
        fi

	#Remove the processed photo name from the list.
        photos_alb_flt=${photos_alb_flt/$photo_fold_flt/}
    done

    # Remove the double space 
    photos_alb_flt=$(echo "$photos_alb_flt" | tr -s ' ')

    # If the list is not empty (for deletion).
    if [ "$photos_alb_flt" != " " ] ; then
        # Print the name of the photo we are going to get rid off
        echo "Photo(s) to delete: $photos_alb_flt."

        # Get the list of all the photos' titles and IDs 
        photos_id_name=$(curl -X POST -H "Authorization: $token" \
        -H 'Content-Type: application/json' -H 'Accept: application/json' \
        -d "{\"albumID\": $id_album}" \
        https://photoserver.mde.epf.fr/api/Album::get | jq '.photos[] | .id, .title')
        photos_id_name=$(echo $photos_id_name | tr -d '"')

           # a loop for each photo in the list to delete
        for photos_remove in $photos_alb_flt ; do
            photo_rm_id=""
            # a loop for each photo in the list 
            for photo_present in $photos_id_name ; do
                # If the name of the photo is found 
                if [ "$photo_present" == "$photos_remove" ] ; then
                    # Leave the loop
                    break
                fi
                # Get previous value to delete the ID of the photo
                photo_rm_id=$photo_present
            done
            # Delete the photo from the album
            curl -X POST -H "Authorization: $token" \
            -H 'Content-Type: application/json' -H 'Accept: application/json' \
            -d "{\"photoIDs\": [\"$photo_rm_id\"]}" \
            https://photoserver.mde.epf.fr/api/Photo::delete
        done

        # Print a message indicating that the deletion of the photos has been completed.
        echo "Photo Deleted"
    else
        echo "Photos Not Deleted"
    fi

    # Perform the Logout
    curl -X POST -H "Authorization: $token" \
    -H 'Content-Type: application/json' -H 'Accept: application/json' \
    -d "{\"username\": \"$user\", \"password\": \"$password\"}" \
    https://photoserver.mde.epf.fr/api/Session::logout
    echo "Goodbye Have a nice day"
    exit 1
else
    echo "The shared folder is not mounted"
	echo "Goodbye Have a nice day"
    exit 0
fi


# Detailed explaination on how this script works 

#The script sets some variables for authentication purposes: user, password, altpassword, and token. These variables contain the necessary credentials to access the API.
#The script checks if "/home/datasource" is properly mounted. It uses the mount command and checks if the folder is present in the output.
#If the folder is properly mounted, the script connect to the server. It uses the curl command to send a request to the server
#After successfull logging in, the script gets the ID of the album associated with the token. It sends another POST request to the server's albums endpoint, retrieves the response using jq (a tool for processing JSON data), and extracts the ID from the response.
#The script retrieves the names of the photos already present in the album. It sends a POST request to the lychee server
#The script begins processing the photos in the folder. It loops through each file with the ".jpg" extension in order to get only photo and not files such as the pyhton config in the "/home/datasource" folder.
#For each photo in the folder, the script performs the following steps:

#Cleans the photo name by removing the file extension and any unwanted characters.
#Checks if the photo is already present in the album by comparing its name with the list of photo names obtained earlier.
#If the photo is not present in the album, it proceeds with the following operations:
#Extracts metadata from the photo using the exiftool command.
#Uploads the photo to the album using a multipart/form-data request with the curl command, obtaining the photo's ID.
#Launches "getColor.py" to get the colors for the photo.
#Modifies the color format and injects the color as tags for the photo using another POST request to the server's photo endpoint.
#After processing each photo, the script removes the processed photo name from the list of photo names obtained earlier.
# Once it is done, the script checks if the list of the photo names is not empty. If it is is not empty, it is time for the deleting process.
# Finally, the script performs a logout and an exit message to notify the user that tha script is done.
````
