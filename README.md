#!/bin/bash

# welcoming
echo -e "\n\nWELCOME TO OSAMA COMPRESSION TOOL\n\n_______________________________________________________________________________\n\n"

while [ "$quit" != 'q' ]; do
    # declare HashMaps
    declare -A hashmapCompress
    declare -A hashmapDecompress

    # The program asks the user if the dictionary.txt file exist or not
    read -r -p "[+] is there a dictionary.txt file? (y/n): " answer

    # If yes, the program asks the user to enter the path of dictionary.txt, read this path, and load the dictionary into the appropriate data structure.
    if [ "$answer" == "y" ]; then
        read -r -p "[+] Enter the path of dictionary.txt: " dictPath
        while true; do
            if [ -e "$dictPath" ]; then
                break
            else
                echo "[-] The file not exist, please try again"
                read -r -p "[+] Enter the path of dictionary.txt: " dictPath
                continue
            fi
        done
        dict=1

        # load the data from dictionary.txt into hashmapDecompress
        numOfLines=0
        while read -r line; do
            ((numOfLines++))
            key=$(echo "$line" | cut -d' ' -f1)
            value=$(echo "$line" | cut -d' ' -f2)
            hashmapDecompress[$key]=$value
            hashmapCompress[$value]=$key
            echo -e "$key" "$value"
        done <"$dictPath"

    # If no, the program creates a new empty dictionary.txt
    elif [ "$answer" == "n" ]; then
        dict=0
        # create the dictionary.txt file in the current directory
        touch dictionary.txt
        dictPath=dictionary.txt
        echo "[+] dictionary.txt file created"
    fi

    # The program asks the user whether he or she wants to do compression or decompression
    read -r -p "[+] Do you want to do compression or decompression? (c/d): " answer

    # converte the answer to lower case (case insensetive)
    answer=$(echo "$answer" | tr '[:upper:]' '[:lower:]')

    compress=-1

    # check hte answer
    while [ "$compress" -eq -1 ]; do
        case "$answer" in
        'c')
            compress=1
            ;;
        "compress")
            compress=1
            ;;
        "compression")
            compress=1
            ;;
        'd')
            compress=0
            ;;
        "decompress")
            compress=0
            ;;
        "decompression")
            compress=0
            ;;
        *)
            echo "[-] Invalid input, please enter c or d"
            compress=-1
            read -r -p "[+] Do you want to do compression or decompression? (c/d): " answer
            answer=$(echo "$answer" | tr '[:upper:]' '[:lower:]')
            ;;
        esac # end case
    done     # end while()

    if [ "$compress" -eq 1 ]; then

        # In the case of compression
        read -r -p "[+] Enter the path of the file to be compressed: " path
        while true; do
            if [ -e "$path" ]; then
                break
            else
                echo "[-] The file not exist, please try again"
                read -r -p "[+] Enter the path of the file to be compressed: " path
                continue
            fi
        done

        cat "$path" | tr '\n' '#' >temp.txt
        mv temp.txt "$path"
        cat $path | tr ' ' '+' >temp.txt
        mv temp.txt "$path"

        # read the data from text file
        specialChars=$(grep -o "[^A-Za-z]" "$path")
        words=$(cat "$path" | tr -s '[:punct:]' ' ' | cut -d' ' -f1-)

        # if the value doesn't in the hashmapCompress, add it
        j=$numOfLines

        for word in $words; do
            if [ "${hashmapCompress[$word]+exists}" ]; then
                :
            else
                hashmapCompress[$word]=$(printf "0x%04x" $j)
                hashmapDecompress[$(printf "0x%04x" $j)]=$word

                # add to the dictionary.txt
                echo "$(printf "0x%04x" $j) $word" >>$dictPath
                ((j++))
            fi
        done # end for()

        # add special chars

        #add ' '
        if [ "${hashmapCompress[" "]+exists}" ]; then
            :
        else
            hashmapCompress[" "]=$(printf "0x%04x" $j)
            hashmapDecompress[$(printf "0x%04x" $j)]=" "
            echo "$(printf "0x%04x" $j) " >>$dictPath
            ((j++))
        fi
        for char in $specialChars; do
            if [ "${hashmapCompress[$char]+exists}" ]; then
                :
            else
                hashmapCompress[$char]=$(printf "0x%04x" $j)
                hashmapDecompress[$(printf "0x%04x" $j)]=$char

                # add to the dictionary.txt
                echo "$(printf "0x%04x" $j) $char" >>$dictPath
                ((j++))
            fi
        done # end for []

        size=$(($(wc -c <"$path") * 16 / 8))

        # compress the text file
        tmpStr=""
        while IFS= read -r -n1 char; do
    
            # if char is not alphapetic char
            if [[ ! "$char" =~ [A-Za-z] ]]; then
                # if tmpStr is not empty
                if [ ! -z "$tmpStr" ]; then
                    echo ${hashmapCompress[$tmpStr]} >>compressed.txt
                    tmpStr=""
                fi

                echo ${hashmapCompress[$char]} >>compressed.txt
                tmpStr=""
            else
                tmpStr+="$char"
            fi

        done <$path
        mv compressed.txt $path

        # claculate the sizes and ratio
        echo "[+] The size of uncompressed file is: $size Bytes"
        size1=$(($(wc -l <"$path") * 16 / 8))
        echo "[+] The size of compressed file is: $size1 Bytes"
        ratio=$(echo "scale=2; $size / $size1" | bc)
        echo "[+] File compression ratio $ratio"
    else
        if [ $dict -eq 0 ]; then
            echo "[-] There is no dictionary to decompress any file, please try again"
            exit 1
        fi
        # In the case of decompression
        read -p "[+] Enter the path of the file to be decompressed: " path
        while true; do
            if [ -e $path ]; then
                break
            else
                echo "[-] The file not exist, please try again"
                read -p "[+] Enter the path of the file to be decompressed: " path
                continue
            fi
        done
        # read line by line from compressed.txt and write on decompressed.txt
        while read line; do
            if [ "${hashmapDecompress[$line]+exist}" ]; then
                : #pass

            else
                echo "[-] There is a code in the file, does not exist in the dictionary file"
                exit 1
            fi
            if [ "${hashmapDecompress[$line]}" = "#" ]; then
                echo -e "" >>decompressed.txt
                continue
            elif [ "${hashmapDecompress[$line]}" = "+" ]; then
                echo -n " " >>decompressed.txt
                continue
            fi # end of if []

            echo -n ${hashmapDecompress[$line]} >>decompressed.txt
        done <$path
        mv decompressed.txt $path
        echo "[+] The file decompressed succesfully"
    fi # end big if []

    read -r -p "[+] To quit enter 'q' or enter anything to continue: " quit
done

echo -e "\nGood Bye!\n"
